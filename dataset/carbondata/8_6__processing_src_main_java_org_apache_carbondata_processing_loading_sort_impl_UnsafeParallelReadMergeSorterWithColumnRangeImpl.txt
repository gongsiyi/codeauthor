1:f82b10b: /*
1:f82b10b:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:f82b10b:  * contributor license agreements.  See the NOTICE file distributed with
1:f82b10b:  * this work for additional information regarding copyright ownership.
1:f82b10b:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:f82b10b:  * (the "License"); you may not use this file except in compliance with
1:f82b10b:  * the License.  You may obtain a copy of the License at
1:f82b10b:  *
1:f82b10b:  *    http://www.apache.org/licenses/LICENSE-2.0
1:f82b10b:  *
1:f82b10b:  * Unless required by applicable law or agreed to in writing, software
1:f82b10b:  * distributed under the License is distributed on an "AS IS" BASIS,
1:f82b10b:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:f82b10b:  * See the License for the specific language governing permissions and
1:f82b10b:  * limitations under the License.
1:f82b10b:  */
7:f82b10b: 
1:349c59c: package org.apache.carbondata.processing.loading.sort.impl;
1:f82b10b: 
1:f82b10b: import java.io.File;
1:d5396b1: import java.util.ArrayList;
1:f82b10b: import java.util.Iterator;
1:f82b10b: import java.util.List;
1:f82b10b: import java.util.concurrent.ExecutorService;
1:f82b10b: import java.util.concurrent.Executors;
1:f82b10b: import java.util.concurrent.TimeUnit;
1:d5396b1: import java.util.concurrent.atomic.AtomicLong;
1:f82b10b: 
1:f82b10b: import org.apache.carbondata.common.CarbonIterator;
1:f82b10b: import org.apache.carbondata.common.logging.LogService;
1:f82b10b: import org.apache.carbondata.common.logging.LogServiceFactory;
1:f82b10b: import org.apache.carbondata.core.constants.CarbonCommonConstants;
1:dc83b2a: import org.apache.carbondata.core.datastore.row.CarbonRow;
1:d5396b1: import org.apache.carbondata.core.metadata.schema.ColumnRangeInfo;
1:f82b10b: import org.apache.carbondata.core.util.CarbonProperties;
1:f82b10b: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
1:349c59c: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1:349c59c: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1:349c59c: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeCarbonRowPage;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeSortDataRows;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeIntermediateMerger;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
1:f82b10b: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1:f82b10b: 
1:d5396b1: import org.apache.commons.lang3.StringUtils;
1:d5396b1: 
1:f82b10b: /**
1:f82b10b:  * It parallely reads data from array of iterates and do merge sort.
1:f82b10b:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:f82b10b:  * final merge sort result.
1:d5396b1:  * This step is specifically for the data loading with specifying column value range, such as
1:d5396b1:  * bucketing, sort_column_bounds, it sorts each range of data separately and write to temp files.
1:f82b10b:  */
1:d5396b1: public class UnsafeParallelReadMergeSorterWithColumnRangeImpl extends AbstractMergeSorter {
1:f82b10b: 
1:f82b10b:   private static final LogService LOGGER =
1:be05695:       LogServiceFactory.getLogService(
1:d5396b1:           UnsafeParallelReadMergeSorterWithColumnRangeImpl.class.getName());
1:f82b10b: 
1:d5396b1:   private SortParameters originSortParameters;
1:d5396b1:   private UnsafeIntermediateMerger[] intermediateFileMergers;
1:d5396b1:   private int inMemoryChunkSizeInMB;
1:d5396b1:   private AtomicLong rowCounter;
1:d5396b1:   private ColumnRangeInfo columnRangeInfo;
1:d5396b1:   /**
1:d5396b1:    * counters to collect information about rows processed by each range
1:d5396b1:    */
1:d5396b1:   private List<AtomicLong> insideRowCounterList;
1:f82b10b: 
1:d5396b1:   public UnsafeParallelReadMergeSorterWithColumnRangeImpl(AtomicLong rowCounter,
1:d5396b1:       ColumnRangeInfo columnRangeInfo) {
1:d5396b1:     this.rowCounter = rowCounter;
1:d5396b1:     this.columnRangeInfo = columnRangeInfo;
4:f82b10b:   }
1:f82b10b: 
1:f82b10b:   @Override public void initialize(SortParameters sortParameters) {
1:d5396b1:     this.originSortParameters = sortParameters;
1:d5396b1:     int totalInMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
1:d5396b1:     inMemoryChunkSizeInMB = totalInMemoryChunkSizeInMB / columnRangeInfo.getNumOfRanges();
1:d5396b1:     if (inMemoryChunkSizeInMB < 5) {
1:d5396b1:       inMemoryChunkSizeInMB = 5;
1:d5396b1:     }
1:d5396b1:     this.insideRowCounterList = new ArrayList<>(columnRangeInfo.getNumOfRanges());
1:d5396b1:     for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:d5396b1:       insideRowCounterList.add(new AtomicLong(0));
1:d5396b1:     }
1:f82b10b:   }
1:f82b10b: 
1:f82b10b:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:f82b10b:       throws CarbonDataLoadingException {
1:d5396b1:     UnsafeSortDataRows[] sortDataRows = new UnsafeSortDataRows[columnRangeInfo.getNumOfRanges()];
1:d5396b1:     intermediateFileMergers = new UnsafeIntermediateMerger[columnRangeInfo.getNumOfRanges()];
1:d5396b1:     SortParameters[] sortParameterArray = new SortParameters[columnRangeInfo.getNumOfRanges()];
1:f82b10b:     try {
1:d5396b1:       for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:d5396b1:         SortParameters parameters = originSortParameters.getCopy();
1:f82b10b:         parameters.setPartitionID(i + "");
1:d5396b1:         parameters.setRangeId(i);
1:d5396b1:         sortParameterArray[i] = parameters;
1:f82b10b:         setTempLocation(parameters);
1:f82b10b:         intermediateFileMergers[i] = new UnsafeIntermediateMerger(parameters);
1:f82b10b:         sortDataRows[i] =
1:f82b10b:             new UnsafeSortDataRows(parameters, intermediateFileMergers[i], inMemoryChunkSizeInMB);
1:f82b10b:         sortDataRows[i].initialize();
1:f82b10b:       }
1:b439b00:     } catch (Exception e) {
2:f82b10b:       throw new CarbonDataLoadingException(e);
1:f82b10b:     }
1:dc83b2a:     ExecutorService executorService = Executors.newFixedThreadPool(iterators.length);
1:06b0d08:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
1:f82b10b:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
1:f82b10b:     try {
1:f82b10b:       for (int i = 0; i < iterators.length; i++) {
1:d5396b1:         executorService.execute(new SortIteratorThread(iterators[i], sortDataRows, rowCounter,
1:d5396b1:             this.insideRowCounterList, this.threadStatusObserver));
1:f82b10b:       }
1:f82b10b:       executorService.shutdown();
1:f82b10b:       executorService.awaitTermination(2, TimeUnit.DAYS);
1:d5396b1:       processRowToNextStep(sortDataRows, originSortParameters);
1:f82b10b:     } catch (Exception e) {
1:06b0d08:       checkError();
1:f82b10b:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:f82b10b:     }
1:06b0d08:     checkError();
1:f82b10b:     try {
1:f82b10b:       for (int i = 0; i < intermediateFileMergers.length; i++) {
1:f82b10b:         intermediateFileMergers[i].finish();
1:f82b10b:       }
1:f82b10b:     } catch (Exception e) {
1:f82b10b:       throw new CarbonDataLoadingException(e);
1:f82b10b:     }
1:f82b10b: 
1:d5396b1:     Iterator<CarbonRowBatch>[] batchIterator = new Iterator[columnRangeInfo.getNumOfRanges()];
1:f82b10b:     for (int i = 0; i < sortDataRows.length; i++) {
1:d5396b1:       batchIterator[i] =
1:d5396b1:           new MergedDataIterator(sortParameterArray[i], batchSize, intermediateFileMergers[i]);
1:f82b10b:     }
1:f82b10b: 
1:f82b10b:     return batchIterator;
1:f82b10b:   }
1:f82b10b: 
1:d5396b1:   private UnsafeSingleThreadFinalSortFilesMerger getFinalMerger(SortParameters sortParameters) {
1:d5396b1:     String[] storeLocation = CarbonDataProcessorUtil
1:d5396b1:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
1:d5396b1:             String.valueOf(sortParameters.getTaskNo()),
1:d5396b1:             sortParameters.getSegmentId() + "", false, false);
1:f82b10b:     // Set the data file location
1:ded8b41:     String[] dataFolderLocation = CarbonDataProcessorUtil.arrayAppend(storeLocation,
1:ded8b41:         File.separator, CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:f82b10b:     return new UnsafeSingleThreadFinalSortFilesMerger(sortParameters, dataFolderLocation);
1:f82b10b:   }
1:f82b10b: 
1:f82b10b:   @Override public void close() {
1:d5396b1:     for (int i = 0; i < intermediateFileMergers.length; i++) {
1:d5396b1:       intermediateFileMergers[i].close();
1:d5396b1:     }
1:f82b10b:   }
1:f82b10b: 
1:f82b10b:   /**
1:f82b10b:    * Below method will be used to process data to next step
1:f82b10b:    */
1:f82b10b:   private boolean processRowToNextStep(UnsafeSortDataRows[] sortDataRows, SortParameters parameters)
1:f82b10b:       throws CarbonDataLoadingException {
1:f82b10b:     try {
1:f82b10b:       for (int i = 0; i < sortDataRows.length; i++) {
1:f82b10b:         // start sorting
1:f82b10b:         sortDataRows[i].startSorting();
1:f82b10b:       }
1:f82b10b:       // check any more rows are present
2:f82b10b:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:f82b10b:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:f82b10b:           .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:f82b10b:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:f82b10b:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
2:f82b10b:       return false;
1:f82b10b:     } catch (Exception e) {
1:f82b10b:       throw new CarbonDataLoadingException(e);
1:f82b10b:     }
1:f82b10b:   }
1:f82b10b: 
1:f82b10b:   private void setTempLocation(SortParameters parameters) {
1:ded8b41:     String[] carbonDataDirectoryPath = CarbonDataProcessorUtil
1:f82b10b:         .getLocalDataFolderLocation(parameters.getDatabaseName(), parameters.getTableName(),
1:5bedd77:             parameters.getTaskNo(), parameters.getSegmentId(),
1:5bedd77:             false, false);
1:ded8b41:     String[] tmpLoc = CarbonDataProcessorUtil.arrayAppend(carbonDataDirectoryPath, File.separator,
1:ded8b41:         CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:ca20160:     LOGGER.warn("set temp location: " + StringUtils.join(tmpLoc, ", "));
1:ded8b41:     parameters.setTempFileLocation(tmpLoc);
1:f82b10b:   }
1:f82b10b: 
1:f82b10b:   /**
1:f82b10b:    * This thread iterates the iterator and adds the rows to @{@link UnsafeSortDataRows}
1:f82b10b:    */
1:06b0d08:   private static class SortIteratorThread implements Runnable {
1:f82b10b: 
1:f82b10b:     private Iterator<CarbonRowBatch> iterator;
1:f82b10b: 
1:f82b10b:     private UnsafeSortDataRows[] sortDataRows;
1:d5396b1:     private AtomicLong rowCounter;
1:d5396b1:     private List<AtomicLong> insideRowCounterList;
1:06b0d08:     private ThreadStatusObserver threadStatusObserver;
1:06b0d08: 
1:f82b10b:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator,
1:d5396b1:         UnsafeSortDataRows[] sortDataRows, AtomicLong rowCounter,
1:d5396b1:         List<AtomicLong> insideRowCounterList,
1:d5396b1:         ThreadStatusObserver threadStatusObserver) {
1:f82b10b:       this.iterator = iterator;
1:f82b10b:       this.sortDataRows = sortDataRows;
1:d5396b1:       this.rowCounter = rowCounter;
1:d5396b1:       this.insideRowCounterList = insideRowCounterList;
1:06b0d08:       this.threadStatusObserver = threadStatusObserver;
1:f82b10b:     }
1:f82b10b: 
1:06b0d08:     @Override
1:06b0d08:     public void run() {
1:f82b10b:       try {
1:f82b10b:         while (iterator.hasNext()) {
1:f82b10b:           CarbonRowBatch batch = iterator.next();
1:f82b10b:           while (batch.hasNext()) {
1:f82b10b:             CarbonRow row = batch.next();
1:f82b10b:             if (row != null) {
1:d5396b1:               UnsafeSortDataRows sortDataRow = sortDataRows[row.getRangeId()];
1:f82b10b:               synchronized (sortDataRow) {
1:d5396b1:                 rowCounter.getAndIncrement();
1:d5396b1:                 insideRowCounterList.get(row.getRangeId()).getAndIncrement();
1:f82b10b:                 sortDataRow.addRow(row.getData());
1:f82b10b:               }
1:f82b10b:             }
1:f82b10b:           }
1:f82b10b:         }
1:d5396b1:         LOGGER.info("Rows processed by each range: " + insideRowCounterList);
1:f82b10b:       } catch (Exception e) {
1:f82b10b:         LOGGER.error(e);
1:06b0d08:         this.threadStatusObserver.notifyFailed(e);
1:f82b10b:       }
1:f82b10b:     }
1:f82b10b:   }
1:f82b10b: 
1:f82b10b:   private class MergedDataIterator extends CarbonIterator<CarbonRowBatch> {
1:f82b10b: 
1:d5396b1:     private SortParameters sortParameters;
1:f82b10b: 
1:f82b10b:     private int batchSize;
1:f82b10b: 
1:f82b10b:     private boolean firstRow;
1:f82b10b: 
1:f82b10b:     private UnsafeIntermediateMerger intermediateMerger;
1:f82b10b: 
1:d5396b1:     public MergedDataIterator(SortParameters sortParameters, int batchSize,
1:f82b10b:         UnsafeIntermediateMerger intermediateMerger) {
1:d5396b1:       this.sortParameters = sortParameters;
1:f82b10b:       this.batchSize = batchSize;
1:f82b10b:       this.intermediateMerger = intermediateMerger;
1:f82b10b:       this.firstRow = true;
1:f82b10b:     }
1:f82b10b: 
1:f82b10b:     private UnsafeSingleThreadFinalSortFilesMerger finalMerger;
1:f82b10b: 
1:f82b10b:     @Override public boolean hasNext() {
1:f82b10b:       if (firstRow) {
1:f82b10b:         firstRow = false;
1:d5396b1:         finalMerger = getFinalMerger(sortParameters);
1:f82b10b:         List<UnsafeCarbonRowPage> rowPages = intermediateMerger.getRowPages();
1:f82b10b:         finalMerger.startFinalMerge(rowPages.toArray(new UnsafeCarbonRowPage[rowPages.size()]),
1:f82b10b:             intermediateMerger.getMergedPages());
1:f82b10b:       }
1:f82b10b:       return finalMerger.hasNext();
1:f82b10b:     }
1:f82b10b: 
1:f82b10b:     @Override public CarbonRowBatch next() {
1:f82b10b:       int counter = 0;
1:f82b10b:       CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
1:f82b10b:       while (finalMerger.hasNext() && counter < batchSize) {
1:f82b10b:         rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:f82b10b:         counter++;
1:f82b10b:       }
1:f82b10b:       return rowBatch;
1:d5396b1:     }
1:f82b10b:   }
1:f82b10b: }
============================================================================
author:sraghunandan
-------------------------------------------------------------------------------
commit:f911403
/////////////////////////////////////////////////////////////////////////
author:ndwangsen
-------------------------------------------------------------------------------
commit:ca20160
/////////////////////////////////////////////////////////////////////////
1:     LOGGER.warn("set temp location: " + StringUtils.join(tmpLoc, ", "));
author:xuchuanyin
-------------------------------------------------------------------------------
commit:c471386
/////////////////////////////////////////////////////////////////////////
commit:b439b00
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:     } catch (Exception e) {
commit:d5396b1
/////////////////////////////////////////////////////////////////////////
1: import java.util.ArrayList;
1: import java.util.concurrent.atomic.AtomicLong;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.metadata.schema.ColumnRangeInfo;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.commons.lang3.StringUtils;
1: 
1:  * This step is specifically for the data loading with specifying column value range, such as
1:  * bucketing, sort_column_bounds, it sorts each range of data separately and write to temp files.
1: public class UnsafeParallelReadMergeSorterWithColumnRangeImpl extends AbstractMergeSorter {
1:           UnsafeParallelReadMergeSorterWithColumnRangeImpl.class.getName());
1:   private SortParameters originSortParameters;
1:   private UnsafeIntermediateMerger[] intermediateFileMergers;
1:   private int inMemoryChunkSizeInMB;
1:   private AtomicLong rowCounter;
1:   private ColumnRangeInfo columnRangeInfo;
1:   /**
1:    * counters to collect information about rows processed by each range
1:    */
1:   private List<AtomicLong> insideRowCounterList;
1:   public UnsafeParallelReadMergeSorterWithColumnRangeImpl(AtomicLong rowCounter,
1:       ColumnRangeInfo columnRangeInfo) {
1:     this.rowCounter = rowCounter;
1:     this.columnRangeInfo = columnRangeInfo;
1:     this.originSortParameters = sortParameters;
1:     int totalInMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
1:     inMemoryChunkSizeInMB = totalInMemoryChunkSizeInMB / columnRangeInfo.getNumOfRanges();
1:     if (inMemoryChunkSizeInMB < 5) {
1:       inMemoryChunkSizeInMB = 5;
1:     }
1:     this.insideRowCounterList = new ArrayList<>(columnRangeInfo.getNumOfRanges());
1:     for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:       insideRowCounterList.add(new AtomicLong(0));
1:     }
1:     UnsafeSortDataRows[] sortDataRows = new UnsafeSortDataRows[columnRangeInfo.getNumOfRanges()];
1:     intermediateFileMergers = new UnsafeIntermediateMerger[columnRangeInfo.getNumOfRanges()];
1:     SortParameters[] sortParameterArray = new SortParameters[columnRangeInfo.getNumOfRanges()];
1:       for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:         SortParameters parameters = originSortParameters.getCopy();
1:         parameters.setRangeId(i);
1:         sortParameterArray[i] = parameters;
/////////////////////////////////////////////////////////////////////////
1:         executorService.execute(new SortIteratorThread(iterators[i], sortDataRows, rowCounter,
1:             this.insideRowCounterList, this.threadStatusObserver));
1:       processRowToNextStep(sortDataRows, originSortParameters);
/////////////////////////////////////////////////////////////////////////
1:     Iterator<CarbonRowBatch>[] batchIterator = new Iterator[columnRangeInfo.getNumOfRanges()];
1:       batchIterator[i] =
1:           new MergedDataIterator(sortParameterArray[i], batchSize, intermediateFileMergers[i]);
1:   private UnsafeSingleThreadFinalSortFilesMerger getFinalMerger(SortParameters sortParameters) {
1:     String[] storeLocation = CarbonDataProcessorUtil
1:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
1:             String.valueOf(sortParameters.getTaskNo()),
1:             sortParameters.getSegmentId() + "", false, false);
/////////////////////////////////////////////////////////////////////////
1:     for (int i = 0; i < intermediateFileMergers.length; i++) {
1:       intermediateFileMergers[i].close();
1:     }
/////////////////////////////////////////////////////////////////////////
0:     LOGGER.error("set temp location: " + StringUtils.join(tmpLoc, ", "));
/////////////////////////////////////////////////////////////////////////
1:     private AtomicLong rowCounter;
1:     private List<AtomicLong> insideRowCounterList;
1:         UnsafeSortDataRows[] sortDataRows, AtomicLong rowCounter,
1:         List<AtomicLong> insideRowCounterList,
1:         ThreadStatusObserver threadStatusObserver) {
1:       this.rowCounter = rowCounter;
1:       this.insideRowCounterList = insideRowCounterList;
/////////////////////////////////////////////////////////////////////////
1:               UnsafeSortDataRows sortDataRow = sortDataRows[row.getRangeId()];
1:                 rowCounter.getAndIncrement();
1:                 insideRowCounterList.get(row.getRangeId()).getAndIncrement();
1:         LOGGER.info("Rows processed by each range: " + insideRowCounterList);
1:     private SortParameters sortParameters;
/////////////////////////////////////////////////////////////////////////
1:     public MergedDataIterator(SortParameters sortParameters, int batchSize,
1:       this.sortParameters = sortParameters;
/////////////////////////////////////////////////////////////////////////
1:         finalMerger = getFinalMerger(sortParameters);
/////////////////////////////////////////////////////////////////////////
1: }
commit:ded8b41
/////////////////////////////////////////////////////////////////////////
0:     String[] storeLocation = CarbonDataProcessorUtil
1:     String[] dataFolderLocation = CarbonDataProcessorUtil.arrayAppend(storeLocation,
1:         File.separator, CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
/////////////////////////////////////////////////////////////////////////
1:     String[] carbonDataDirectoryPath = CarbonDataProcessorUtil
1:     String[] tmpLoc = CarbonDataProcessorUtil.arrayAppend(carbonDataDirectoryPath, File.separator,
1:         CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:     parameters.setTempFileLocation(tmpLoc);
commit:be05695
/////////////////////////////////////////////////////////////////////////
1:       LogServiceFactory.getLogService(
0:                 UnsafeParallelReadMergeSorterWithBucketingImpl.class.getName());
author:Jacky Li
-------------------------------------------------------------------------------
commit:5bedd77
/////////////////////////////////////////////////////////////////////////
0:       batchIterator[i] = new MergedDataIterator(batchSize, intermediateFileMergers[i]);
0:   private UnsafeSingleThreadFinalSortFilesMerger getFinalMerger() {
0:     String[] storeLocation = CarbonDataProcessorUtil.getLocalDataFolderLocation(
0:         sortParameters.getDatabaseName(), sortParameters.getTableName(),
0:         String.valueOf(sortParameters.getTaskNo()), sortParameters.getSegmentId(),
1:         false, false);
/////////////////////////////////////////////////////////////////////////
1:             parameters.getTaskNo(), parameters.getSegmentId(),
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0:     public MergedDataIterator(int batchSize,
/////////////////////////////////////////////////////////////////////////
0:         finalMerger = getFinalMerger();
commit:349c59c
/////////////////////////////////////////////////////////////////////////
1: package org.apache.carbondata.processing.loading.sort.impl;
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.processing.loading.DataField;
1: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeCarbonRowPage;
1: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeSortDataRows;
1: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeIntermediateMerger;
1: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
1: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
author:Raghunandan S
-------------------------------------------------------------------------------
commit:06b0d08
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.processing.newflow.sort.AbstractMergeSorter;
/////////////////////////////////////////////////////////////////////////
0: public class UnsafeParallelReadMergeSorterWithBucketingImpl extends AbstractMergeSorter {
/////////////////////////////////////////////////////////////////////////
1:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
0:         executorService.execute(new SortIteratorThread(iterators[i], sortDataRows, this
0:             .threadStatusObserver));
1:       checkError();
1:     checkError();
/////////////////////////////////////////////////////////////////////////
1:   private static class SortIteratorThread implements Runnable {
1:     private ThreadStatusObserver threadStatusObserver;
1: 
0:         UnsafeSortDataRows[] sortDataRows, ThreadStatusObserver threadStatusObserver) {
1:       this.threadStatusObserver = threadStatusObserver;
1:     @Override
1:     public void run() {
/////////////////////////////////////////////////////////////////////////
1:         this.threadStatusObserver.notifyFailed(e);
author:lionelcao
-------------------------------------------------------------------------------
commit:874764f
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getSegmentId() + "", false, false);
/////////////////////////////////////////////////////////////////////////
0:             parameters.getTaskNo(), parameters.getPartitionID(), parameters.getSegmentId(),
0:             false, false);
author:jackylk
-------------------------------------------------------------------------------
commit:edda248
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.core.memory.MemoryException;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0:     } catch (MemoryException e) {
commit:dc83b2a
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.datastore.row.CarbonRow;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:     ExecutorService executorService = Executors.newFixedThreadPool(iterators.length);
author:ravipesala
-------------------------------------------------------------------------------
commit:f82b10b
/////////////////////////////////////////////////////////////////////////
1: /*
1:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:  * contributor license agreements.  See the NOTICE file distributed with
1:  * this work for additional information regarding copyright ownership.
1:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:  * (the "License"); you may not use this file except in compliance with
1:  * the License.  You may obtain a copy of the License at
1:  *
1:  *    http://www.apache.org/licenses/LICENSE-2.0
1:  *
1:  * Unless required by applicable law or agreed to in writing, software
1:  * distributed under the License is distributed on an "AS IS" BASIS,
1:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:  * See the License for the specific language governing permissions and
1:  * limitations under the License.
1:  */
1: 
0: package org.apache.carbondata.processing.newflow.sort.impl;
1: 
1: import java.io.File;
1: import java.util.Iterator;
1: import java.util.List;
0: import java.util.concurrent.Callable;
1: import java.util.concurrent.ExecutorService;
1: import java.util.concurrent.Executors;
1: import java.util.concurrent.TimeUnit;
1: 
1: import org.apache.carbondata.common.CarbonIterator;
1: import org.apache.carbondata.common.logging.LogService;
1: import org.apache.carbondata.common.logging.LogServiceFactory;
1: import org.apache.carbondata.core.constants.CarbonCommonConstants;
0: import org.apache.carbondata.core.metadata.schema.BucketingInfo;
1: import org.apache.carbondata.core.util.CarbonProperties;
1: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
0: import org.apache.carbondata.processing.newflow.DataField;
0: import org.apache.carbondata.processing.newflow.exception.CarbonDataLoadingException;
0: import org.apache.carbondata.processing.newflow.row.CarbonRow;
0: import org.apache.carbondata.processing.newflow.row.CarbonRowBatch;
0: import org.apache.carbondata.processing.newflow.sort.Sorter;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.UnsafeCarbonRowPage;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.UnsafeSortDataRows;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.merger.UnsafeIntermediateMerger;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
0: import org.apache.carbondata.processing.sortandgroupby.exception.CarbonSortKeyAndGroupByException;
0: import org.apache.carbondata.processing.sortandgroupby.sortdata.SortParameters;
1: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1: 
1: /**
1:  * It parallely reads data from array of iterates and do merge sort.
1:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:  * final merge sort result.
0:  * This step is specifically for bucketing, it sorts each bucket data separately and write to
0:  * temp files.
1:  */
0: public class UnsafeParallelReadMergeSorterWithBucketingImpl implements Sorter {
1: 
1:   private static final LogService LOGGER =
0:       LogServiceFactory.getLogService(ParallelReadMergeSorterImpl.class.getName());
1: 
0:   private SortParameters sortParameters;
1: 
0:   private ExecutorService executorService;
1: 
0:   private BucketingInfo bucketingInfo;
1: 
0:   private DataField[] inputDataFields;
1: 
0:   public UnsafeParallelReadMergeSorterWithBucketingImpl(DataField[] inputDataFields,
0:       BucketingInfo bucketingInfo) {
0:     this.inputDataFields = inputDataFields;
0:     this.bucketingInfo = bucketingInfo;
1:   }
1: 
1:   @Override public void initialize(SortParameters sortParameters) {
0:     this.sortParameters = sortParameters;
0:     int buffer = Integer.parseInt(CarbonProperties.getInstance()
0:         .getProperty(CarbonCommonConstants.SORT_SIZE, CarbonCommonConstants.SORT_SIZE_DEFAULT_VAL));
1:   }
1: 
1:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:       throws CarbonDataLoadingException {
0:     UnsafeSortDataRows[] sortDataRows = new UnsafeSortDataRows[bucketingInfo.getNumberOfBuckets()];
0:     UnsafeIntermediateMerger[] intermediateFileMergers =
0:         new UnsafeIntermediateMerger[sortDataRows.length];
0:     int inMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
0:     inMemoryChunkSizeInMB = inMemoryChunkSizeInMB / bucketingInfo.getNumberOfBuckets();
0:     if (inMemoryChunkSizeInMB < 5) {
0:       inMemoryChunkSizeInMB = 5;
1:     }
1:     try {
0:       for (int i = 0; i < bucketingInfo.getNumberOfBuckets(); i++) {
0:         SortParameters parameters = sortParameters.getCopy();
1:         parameters.setPartitionID(i + "");
1:         setTempLocation(parameters);
1:         intermediateFileMergers[i] = new UnsafeIntermediateMerger(parameters);
1:         sortDataRows[i] =
1:             new UnsafeSortDataRows(parameters, intermediateFileMergers[i], inMemoryChunkSizeInMB);
1:         sortDataRows[i].initialize();
1:       }
0:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
0:     this.executorService = Executors.newFixedThreadPool(iterators.length);
1:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
1:     try {
1:       for (int i = 0; i < iterators.length; i++) {
0:         executorService.submit(new SortIteratorThread(iterators[i], sortDataRows));
1:       }
1:       executorService.shutdown();
1:       executorService.awaitTermination(2, TimeUnit.DAYS);
0:       processRowToNextStep(sortDataRows, sortParameters);
1:     } catch (Exception e) {
1:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:     }
1:     try {
1:       for (int i = 0; i < intermediateFileMergers.length; i++) {
1:         intermediateFileMergers[i].finish();
1:       }
1:     } catch (Exception e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1: 
0:     Iterator<CarbonRowBatch>[] batchIterator = new Iterator[bucketingInfo.getNumberOfBuckets()];
1:     for (int i = 0; i < sortDataRows.length; i++) {
0:       batchIterator[i] =
0:           new MergedDataIterator(String.valueOf(i), batchSize, intermediateFileMergers[i]);
1:     }
1: 
1:     return batchIterator;
1:   }
1: 
0:   private UnsafeSingleThreadFinalSortFilesMerger getFinalMerger(String bucketId) {
0:     String storeLocation = CarbonDataProcessorUtil
0:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
0:             String.valueOf(sortParameters.getTaskNo()), bucketId,
0:             sortParameters.getSegmentId() + "", false);
1:     // Set the data file location
0:     String dataFolderLocation =
0:         storeLocation + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION;
1:     return new UnsafeSingleThreadFinalSortFilesMerger(sortParameters, dataFolderLocation);
1:   }
1: 
1:   @Override public void close() {
1:   }
1: 
1:   /**
1:    * Below method will be used to process data to next step
1:    */
1:   private boolean processRowToNextStep(UnsafeSortDataRows[] sortDataRows, SortParameters parameters)
1:       throws CarbonDataLoadingException {
0:     if (null == sortDataRows || sortDataRows.length == 0) {
1:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
0:       LOGGER.info("Number of Records was Zero");
0:       String logMessage = "Summary: Carbon Sort Key Step: Read: " + 0 + ": Write: " + 0;
0:       LOGGER.info(logMessage);
1:       return false;
1:     }
1: 
1:     try {
1:       for (int i = 0; i < sortDataRows.length; i++) {
1:         // start sorting
1:         sortDataRows[i].startSorting();
1:       }
1:       // check any more rows are present
1:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:           .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:       return false;
1:     } catch (Exception e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1:   }
1: 
1:   private void setTempLocation(SortParameters parameters) {
0:     String carbonDataDirectoryPath = CarbonDataProcessorUtil
1:         .getLocalDataFolderLocation(parameters.getDatabaseName(), parameters.getTableName(),
0:             parameters.getTaskNo(), parameters.getPartitionID(), parameters.getSegmentId(), false);
0:     parameters.setTempFileLocation(
0:         carbonDataDirectoryPath + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:   }
1: 
1:   /**
1:    * This thread iterates the iterator and adds the rows to @{@link UnsafeSortDataRows}
1:    */
0:   private static class SortIteratorThread implements Callable<Void> {
1: 
1:     private Iterator<CarbonRowBatch> iterator;
1: 
1:     private UnsafeSortDataRows[] sortDataRows;
1: 
1:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator,
0:         UnsafeSortDataRows[] sortDataRows) {
1:       this.iterator = iterator;
1:       this.sortDataRows = sortDataRows;
1:     }
1: 
0:     @Override public Void call() throws CarbonDataLoadingException {
1:       try {
1:         while (iterator.hasNext()) {
1:           CarbonRowBatch batch = iterator.next();
0:           int i = 0;
1:           while (batch.hasNext()) {
1:             CarbonRow row = batch.next();
1:             if (row != null) {
0:               UnsafeSortDataRows sortDataRow = sortDataRows[row.bucketNumber];
1:               synchronized (sortDataRow) {
1:                 sortDataRow.addRow(row.getData());
1:               }
1:             }
1:           }
1:         }
1:       } catch (Exception e) {
1:         LOGGER.error(e);
1:         throw new CarbonDataLoadingException(e);
1:       }
0:       return null;
1:     }
1: 
1:   }
1: 
1:   private class MergedDataIterator extends CarbonIterator<CarbonRowBatch> {
1: 
0:     private String partitionId;
1: 
1:     private int batchSize;
1: 
1:     private boolean firstRow;
1: 
1:     private UnsafeIntermediateMerger intermediateMerger;
1: 
0:     public MergedDataIterator(String partitionId, int batchSize,
1:         UnsafeIntermediateMerger intermediateMerger) {
0:       this.partitionId = partitionId;
1:       this.batchSize = batchSize;
1:       this.intermediateMerger = intermediateMerger;
1:       this.firstRow = true;
1:     }
1: 
1:     private UnsafeSingleThreadFinalSortFilesMerger finalMerger;
1: 
1:     @Override public boolean hasNext() {
1:       if (firstRow) {
1:         firstRow = false;
0:         finalMerger = getFinalMerger(partitionId);
1:         List<UnsafeCarbonRowPage> rowPages = intermediateMerger.getRowPages();
1:         finalMerger.startFinalMerge(rowPages.toArray(new UnsafeCarbonRowPage[rowPages.size()]),
1:             intermediateMerger.getMergedPages());
1:       }
1:       return finalMerger.hasNext();
1:     }
1: 
1:     @Override public CarbonRowBatch next() {
1:       int counter = 0;
1:       CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
1:       while (finalMerger.hasNext() && counter < batchSize) {
1:         rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:         counter++;
1:       }
1:       return rowBatch;
1:     }
1:   }
1: }
============================================================================
1:cbf8797: /*
1:41347d8:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:41347d8:  * contributor license agreements.  See the NOTICE file distributed with
1:41347d8:  * this work for additional information regarding copyright ownership.
1:41347d8:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:41347d8:  * (the "License"); you may not use this file except in compliance with
1:41347d8:  * the License.  You may obtain a copy of the License at
1:cbf8797:  *
1:cbf8797:  *    http://www.apache.org/licenses/LICENSE-2.0
1:cbf8797:  *
1:41347d8:  * Unless required by applicable law or agreed to in writing, software
1:41347d8:  * distributed under the License is distributed on an "AS IS" BASIS,
1:41347d8:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:41347d8:  * See the License for the specific language governing permissions and
1:41347d8:  * limitations under the License.
1:cbf8797:  */
1:349c59c: package org.apache.carbondata.processing.loading.sort.impl;
6:cbf8797: 
1:cbf8797: import java.io.File;
1:d5396b1: import java.util.ArrayList;
1:cbf8797: import java.util.Iterator;
1:d5396b1: import java.util.List;
1:cbf8797: import java.util.concurrent.ExecutorService;
1:cbf8797: import java.util.concurrent.Executors;
1:cbf8797: import java.util.concurrent.TimeUnit;
1:30f575f: import java.util.concurrent.atomic.AtomicLong;
1:cbf8797: 
1:cbf8797: import org.apache.carbondata.common.CarbonIterator;
1:cbf8797: import org.apache.carbondata.common.logging.LogService;
1:cbf8797: import org.apache.carbondata.common.logging.LogServiceFactory;
1:cbf8797: import org.apache.carbondata.core.constants.CarbonCommonConstants;
1:dc83b2a: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1:dc83b2a: import org.apache.carbondata.core.datastore.row.CarbonRow;
1:d5396b1: import org.apache.carbondata.core.metadata.schema.ColumnRangeInfo;
1:cbf8797: import org.apache.carbondata.core.util.CarbonProperties;
1:cbf8797: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
1:349c59c: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1:349c59c: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1:349c59c: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1:349c59c: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SingleThreadFinalSortFilesMerger;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortDataRows;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortIntermediateFileMerger;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
1:cbf8797: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1:cbf8797: 
1:cbf8797: /**
1:cbf8797:  * It parallely reads data from array of iterates and do merge sort.
1:cbf8797:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:cbf8797:  * final merge sort result.
1:d5396b1:  * This step is specifically for the data loading with specifying column value range, such as
1:d5396b1:  * bucketing,sort_column_bounds, it sorts each range of data separately and write to temp files.
1:cbf8797:  */
1:d5396b1: public class ParallelReadMergeSorterWithColumnRangeImpl extends AbstractMergeSorter {
1:d5396b1:   private static final LogService LOGGER = LogServiceFactory.getLogService(
1:d5396b1:       ParallelReadMergeSorterWithColumnRangeImpl.class.getName());
1:cbf8797: 
1:d5396b1:   private SortParameters originSortParameters;
1:cbf8797: 
1:f82b10b:   private SortIntermediateFileMerger[] intermediateFileMergers;
1:f82b10b: 
1:d5396b1:   private ColumnRangeInfo columnRangeInfo;
1:cbf8797: 
1:cbf8797:   private int sortBufferSize;
1:cbf8797: 
1:30f575f:   private AtomicLong rowCounter;
1:d5396b1:   /**
1:d5396b1:    * counters to collect information about rows processed by each range
1:d5396b1:    */
1:d5396b1:   private List<AtomicLong> insideRowCounterList;
1:30f575f: 
1:d5396b1:   public ParallelReadMergeSorterWithColumnRangeImpl(AtomicLong rowCounter,
1:d5396b1:       ColumnRangeInfo columnRangeInfo) {
1:30f575f:     this.rowCounter = rowCounter;
1:d5396b1:     this.columnRangeInfo = columnRangeInfo;
2:cbf8797:   }
1:cbf8797: 
1:d5396b1:   @Override
1:d5396b1:   public void initialize(SortParameters sortParameters) {
1:d5396b1:     this.originSortParameters = sortParameters;
1:cbf8797:     int buffer = Integer.parseInt(CarbonProperties.getInstance()
1:cbf8797:         .getProperty(CarbonCommonConstants.SORT_SIZE, CarbonCommonConstants.SORT_SIZE_DEFAULT_VAL));
1:d5396b1:     sortBufferSize = buffer / columnRangeInfo.getNumOfRanges();
1:cbf8797:     if (sortBufferSize < 100) {
1:cbf8797:       sortBufferSize = 100;
1:cbf8797:     }
1:d5396b1:     this.insideRowCounterList = new ArrayList<>(columnRangeInfo.getNumOfRanges());
1:d5396b1:     for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:d5396b1:       insideRowCounterList.add(new AtomicLong(0));
1:d5396b1:     }
1:cbf8797:   }
1:cbf8797: 
1:cbf8797:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:cbf8797:       throws CarbonDataLoadingException {
1:d5396b1:     SortDataRows[] sortDataRows = new SortDataRows[columnRangeInfo.getNumOfRanges()];
1:d5396b1:     intermediateFileMergers = new SortIntermediateFileMerger[columnRangeInfo.getNumOfRanges()];
1:d5396b1:     SortParameters[] sortParameterArray = new SortParameters[columnRangeInfo.getNumOfRanges()];
1:cbf8797:     try {
1:d5396b1:       for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:d5396b1:         SortParameters parameters = originSortParameters.getCopy();
1:cbf8797:         parameters.setPartitionID(i + "");
1:d5396b1:         parameters.setRangeId(i);
1:d5396b1:         sortParameterArray[i] = parameters;
1:cbf8797:         setTempLocation(parameters);
1:cbf8797:         parameters.setBufferSize(sortBufferSize);
1:f82b10b:         intermediateFileMergers[i] = new SortIntermediateFileMerger(parameters);
1:f82b10b:         sortDataRows[i] = new SortDataRows(parameters, intermediateFileMergers[i]);
1:cbf8797:         sortDataRows[i].initialize();
1:cbf8797:       }
1:cbf8797:     } catch (CarbonSortKeyAndGroupByException e) {
2:cbf8797:       throw new CarbonDataLoadingException(e);
1:cbf8797:     }
1:dc83b2a:     ExecutorService executorService = Executors.newFixedThreadPool(iterators.length);
1:dc83b2a:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
1:cbf8797:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
1:cbf8797:     try {
1:d5396b1:       // dispatch rows to sortDataRows by range id
1:cbf8797:       for (int i = 0; i < iterators.length; i++) {
1:06b0d08:         executorService.execute(new SortIteratorThread(iterators[i], sortDataRows, rowCounter,
1:d5396b1:             this.insideRowCounterList, this.threadStatusObserver));
1:cbf8797:       }
1:cbf8797:       executorService.shutdown();
1:cbf8797:       executorService.awaitTermination(2, TimeUnit.DAYS);
1:d5396b1:       processRowToNextStep(sortDataRows, originSortParameters);
1:cbf8797:     } catch (Exception e) {
1:53accb3:       checkError();
1:cbf8797:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:cbf8797:     }
1:53accb3:     checkError();
1:cbf8797:     try {
1:f82b10b:       for (int i = 0; i < intermediateFileMergers.length; i++) {
1:f82b10b:         intermediateFileMergers[i].finish();
1:f82b10b:       }
1:cbf8797:     } catch (CarbonDataWriterException e) {
1:cbf8797:       throw new CarbonDataLoadingException(e);
1:cbf8797:     } catch (CarbonSortKeyAndGroupByException e) {
1:cbf8797:       throw new CarbonDataLoadingException(e);
1:cbf8797:     }
1:cbf8797: 
1:d5396b1:     Iterator<CarbonRowBatch>[] batchIterator = new Iterator[columnRangeInfo.getNumOfRanges()];
1:d5396b1:     for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:d5396b1:       batchIterator[i] = new MergedDataIterator(sortParameterArray[i], batchSize);
1:cbf8797:     }
1:cbf8797: 
1:cbf8797:     return batchIterator;
1:cbf8797:   }
1:cbf8797: 
1:d5396b1:   private SingleThreadFinalSortFilesMerger getFinalMerger(SortParameters sortParameters) {
1:d5396b1:     String[] storeLocation = CarbonDataProcessorUtil
1:d5396b1:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
1:d5396b1:             String.valueOf(sortParameters.getTaskNo()),
1:d5396b1:             sortParameters.getSegmentId() + "", false, false);
1:cbf8797:     // Set the data file location
1:ded8b41:     String[] dataFolderLocation = CarbonDataProcessorUtil.arrayAppend(storeLocation, File.separator,
1:ded8b41:         CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:81149f6:     return new SingleThreadFinalSortFilesMerger(dataFolderLocation, sortParameters.getTableName(),
1:d5396b1:         sortParameters);
1:cbf8797:   }
1:cbf8797: 
1:cbf8797:   @Override public void close() {
1:f82b10b:     for (int i = 0; i < intermediateFileMergers.length; i++) {
1:f82b10b:       intermediateFileMergers[i].close();
1:f82b10b:     }
1:cbf8797:   }
1:cbf8797: 
1:cbf8797:   /**
1:cbf8797:    * Below method will be used to process data to next step
1:cbf8797:    */
1:cbf8797:   private boolean processRowToNextStep(SortDataRows[] sortDataRows, SortParameters parameters)
1:cbf8797:       throws CarbonDataLoadingException {
1:cbf8797:     try {
1:cbf8797:       for (int i = 0; i < sortDataRows.length; i++) {
1:cbf8797:         // start sorting
1:cbf8797:         sortDataRows[i].startSorting();
1:cbf8797:       }
1:cbf8797:       // check any more rows are present
2:cbf8797:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:cbf8797:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:cbf8797:           .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:cbf8797:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:cbf8797:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
2:cbf8797:       return false;
1:cbf8797:     } catch (CarbonSortKeyAndGroupByException e) {
1:cbf8797:       throw new CarbonDataLoadingException(e);
1:cbf8797:     }
1:cbf8797:   }
1:cbf8797: 
1:cbf8797:   private void setTempLocation(SortParameters parameters) {
1:d5396b1:     String[] carbonDataDirectoryPath = CarbonDataProcessorUtil
1:d5396b1:         .getLocalDataFolderLocation(parameters.getDatabaseName(),
1:d5396b1:             parameters.getTableName(), parameters.getTaskNo(),
1:d5396b1:             parameters.getSegmentId(), false, false);
1:ded8b41:     String[] tmpLocs = CarbonDataProcessorUtil.arrayAppend(carbonDataDirectoryPath, File.separator,
1:ded8b41:         CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:ded8b41:     parameters.setTempFileLocation(tmpLocs);
1:cbf8797:   }
1:cbf8797: 
1:cbf8797:   /**
1:cbf8797:    * This thread iterates the iterator and adds the rows to @{@link SortDataRows}
1:cbf8797:    */
1:06b0d08:   private static class SortIteratorThread implements Runnable {
1:cbf8797: 
1:cbf8797:     private Iterator<CarbonRowBatch> iterator;
1:cbf8797: 
1:cbf8797:     private SortDataRows[] sortDataRows;
1:30f575f: 
1:30f575f:     private AtomicLong rowCounter;
1:d5396b1:     private List<AtomicLong> insideCounterList;
1:53accb3:     private ThreadStatusObserver threadStatusObserver;
1:cbf8797: 
1:30f575f:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortDataRows[] sortDataRows,
1:d5396b1:         AtomicLong rowCounter, List<AtomicLong> insideCounterList,
1:d5396b1:         ThreadStatusObserver observer) {
1:cbf8797:       this.iterator = iterator;
1:cbf8797:       this.sortDataRows = sortDataRows;
1:30f575f:       this.rowCounter = rowCounter;
1:d5396b1:       this.insideCounterList = insideCounterList;
1:53accb3:       this.threadStatusObserver = observer;
1:cbf8797:     }
1:cbf8797: 
1:06b0d08:     @Override
1:06b0d08:     public void run() {
1:cbf8797:       try {
1:cbf8797:         while (iterator.hasNext()) {
1:cbf8797:           CarbonRowBatch batch = iterator.next();
1:c5aba5f:           while (batch.hasNext()) {
1:c5aba5f:             CarbonRow row = batch.next();
1:cbf8797:             if (row != null) {
1:d5396b1:               SortDataRows sortDataRow = sortDataRows[row.getRangeId()];
1:cbf8797:               synchronized (sortDataRow) {
1:cbf8797:                 sortDataRow.addRow(row.getData());
1:d5396b1:                 insideCounterList.get(row.getRangeId()).getAndIncrement();
1:30f575f:                 rowCounter.getAndAdd(1);
1:cbf8797:               }
1:cbf8797:             }
1:cbf8797:           }
1:cbf8797:         }
1:d5396b1:         LOGGER.info("Rows processed by each range: " + insideCounterList);
1:cbf8797:       } catch (Exception e) {
1:cbf8797:         LOGGER.error(e);
1:53accb3:         this.threadStatusObserver.notifyFailed(e);
1:cbf8797:       }
1:cbf8797:     }
1:cbf8797: 
1:cbf8797:   }
1:cbf8797: 
1:cbf8797:   private class MergedDataIterator extends CarbonIterator<CarbonRowBatch> {
1:cbf8797: 
1:d5396b1:     private SortParameters sortParameters;
1:cbf8797: 
1:cbf8797:     private int batchSize;
1:cbf8797: 
1:cbf8797:     private boolean firstRow = true;
1:cbf8797: 
1:d5396b1:     public MergedDataIterator(SortParameters sortParameters, int batchSize) {
1:d5396b1:       this.sortParameters = sortParameters;
1:cbf8797:       this.batchSize = batchSize;
1:cbf8797:     }
1:cbf8797: 
1:cbf8797:     private SingleThreadFinalSortFilesMerger finalMerger;
1:cbf8797: 
1:cbf8797:     @Override public boolean hasNext() {
1:cbf8797:       if (firstRow) {
1:cbf8797:         firstRow = false;
1:d5396b1:         finalMerger = getFinalMerger(sortParameters);
1:cbf8797:         finalMerger.startFinalMerge();
1:cbf8797:       }
1:cbf8797:       return finalMerger.hasNext();
1:cbf8797:     }
1:cbf8797: 
1:cbf8797:     @Override public CarbonRowBatch next() {
1:cbf8797:       int counter = 0;
1:c5aba5f:       CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
1:cbf8797:       while (finalMerger.hasNext() && counter < batchSize) {
1:cbf8797:         rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:cbf8797:         counter++;
1:cbf8797:       }
1:cbf8797:       return rowBatch;
1:cbf8797:     }
1:cbf8797:   }
1:cbf8797: }
============================================================================
author:sraghunandan
-------------------------------------------------------------------------------
commit:f911403
/////////////////////////////////////////////////////////////////////////
author:xuchuanyin
-------------------------------------------------------------------------------
commit:d5396b1
/////////////////////////////////////////////////////////////////////////
1: import java.util.ArrayList;
1: import java.util.List;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.metadata.schema.ColumnRangeInfo;
/////////////////////////////////////////////////////////////////////////
1:  * This step is specifically for the data loading with specifying column value range, such as
1:  * bucketing,sort_column_bounds, it sorts each range of data separately and write to temp files.
1: public class ParallelReadMergeSorterWithColumnRangeImpl extends AbstractMergeSorter {
1:   private static final LogService LOGGER = LogServiceFactory.getLogService(
1:       ParallelReadMergeSorterWithColumnRangeImpl.class.getName());
1:   private SortParameters originSortParameters;
1:   private ColumnRangeInfo columnRangeInfo;
1:   /**
1:    * counters to collect information about rows processed by each range
1:    */
1:   private List<AtomicLong> insideRowCounterList;
1:   public ParallelReadMergeSorterWithColumnRangeImpl(AtomicLong rowCounter,
1:       ColumnRangeInfo columnRangeInfo) {
1:     this.columnRangeInfo = columnRangeInfo;
1:   @Override
1:   public void initialize(SortParameters sortParameters) {
1:     this.originSortParameters = sortParameters;
1:     sortBufferSize = buffer / columnRangeInfo.getNumOfRanges();
1:     this.insideRowCounterList = new ArrayList<>(columnRangeInfo.getNumOfRanges());
1:     for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:       insideRowCounterList.add(new AtomicLong(0));
1:     }
1:     SortDataRows[] sortDataRows = new SortDataRows[columnRangeInfo.getNumOfRanges()];
1:     intermediateFileMergers = new SortIntermediateFileMerger[columnRangeInfo.getNumOfRanges()];
1:     SortParameters[] sortParameterArray = new SortParameters[columnRangeInfo.getNumOfRanges()];
1:       for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:         SortParameters parameters = originSortParameters.getCopy();
1:         parameters.setRangeId(i);
1:         sortParameterArray[i] = parameters;
/////////////////////////////////////////////////////////////////////////
1:       // dispatch rows to sortDataRows by range id
1:             this.insideRowCounterList, this.threadStatusObserver));
1:       processRowToNextStep(sortDataRows, originSortParameters);
/////////////////////////////////////////////////////////////////////////
1:     Iterator<CarbonRowBatch>[] batchIterator = new Iterator[columnRangeInfo.getNumOfRanges()];
1:     for (int i = 0; i < columnRangeInfo.getNumOfRanges(); i++) {
1:       batchIterator[i] = new MergedDataIterator(sortParameterArray[i], batchSize);
1:   private SingleThreadFinalSortFilesMerger getFinalMerger(SortParameters sortParameters) {
1:     String[] storeLocation = CarbonDataProcessorUtil
1:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
1:             String.valueOf(sortParameters.getTaskNo()),
1:             sortParameters.getSegmentId() + "", false, false);
1:         sortParameters);
/////////////////////////////////////////////////////////////////////////
1:     String[] carbonDataDirectoryPath = CarbonDataProcessorUtil
1:         .getLocalDataFolderLocation(parameters.getDatabaseName(),
1:             parameters.getTableName(), parameters.getTaskNo(),
1:             parameters.getSegmentId(), false, false);
/////////////////////////////////////////////////////////////////////////
1:     private List<AtomicLong> insideCounterList;
1:         AtomicLong rowCounter, List<AtomicLong> insideCounterList,
1:         ThreadStatusObserver observer) {
1:       this.insideCounterList = insideCounterList;
/////////////////////////////////////////////////////////////////////////
1:               SortDataRows sortDataRow = sortDataRows[row.getRangeId()];
1:                 insideCounterList.get(row.getRangeId()).getAndIncrement();
1:         LOGGER.info("Rows processed by each range: " + insideCounterList);
/////////////////////////////////////////////////////////////////////////
1:     private SortParameters sortParameters;
1:     public MergedDataIterator(SortParameters sortParameters, int batchSize) {
1:       this.sortParameters = sortParameters;
/////////////////////////////////////////////////////////////////////////
1:         finalMerger = getFinalMerger(sortParameters);
commit:c100251
/////////////////////////////////////////////////////////////////////////
0:             sortParameters);
commit:ded8b41
/////////////////////////////////////////////////////////////////////////
0:     String[] storeLocation = CarbonDataProcessorUtil
1:     String[] dataFolderLocation = CarbonDataProcessorUtil.arrayAppend(storeLocation, File.separator,
1:         CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
/////////////////////////////////////////////////////////////////////////
0:     String[] carbonDataDirectoryPath = CarbonDataProcessorUtil
1:     String[] tmpLocs = CarbonDataProcessorUtil.arrayAppend(carbonDataDirectoryPath, File.separator,
1:         CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:     parameters.setTempFileLocation(tmpLocs);
commit:be05695
/////////////////////////////////////////////////////////////////////////
0:       LogServiceFactory.getLogService(ParallelReadMergeSorterWithBucketingImpl.class.getName());
author:Jacky Li
-------------------------------------------------------------------------------
commit:5bedd77
/////////////////////////////////////////////////////////////////////////
0:     String[] storeLocation = CarbonDataProcessorUtil.getLocalDataFolderLocation(
0:         sortParameters.getDatabaseName(), sortParameters.getTableName(),
0:         String.valueOf(sortParameters.getTaskNo()), sortParameters.getSegmentId(),
0:         false, false);
/////////////////////////////////////////////////////////////////////////
0:     String[] carbonDataDirectoryPath = CarbonDataProcessorUtil.getLocalDataFolderLocation(
0:         parameters.getDatabaseName(), parameters.getTableName(), parameters.getTaskNo(),
0:         parameters.getSegmentId(), false, false);
commit:349c59c
/////////////////////////////////////////////////////////////////////////
1: package org.apache.carbondata.processing.loading.sort.impl;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1: import org.apache.carbondata.processing.sort.sortdata.SingleThreadFinalSortFilesMerger;
1: import org.apache.carbondata.processing.sort.sortdata.SortDataRows;
1: import org.apache.carbondata.processing.sort.sortdata.SortIntermediateFileMerger;
1: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
author:Raghunandan S
-------------------------------------------------------------------------------
commit:06b0d08
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:         executorService.execute(new SortIteratorThread(iterators[i], sortDataRows, rowCounter,
/////////////////////////////////////////////////////////////////////////
1:   private static class SortIteratorThread implements Runnable {
/////////////////////////////////////////////////////////////////////////
1:     @Override
1:     public void run() {
/////////////////////////////////////////////////////////////////////////
author:lionelcao
-------------------------------------------------------------------------------
commit:874764f
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getSegmentId() + "", false, false);
/////////////////////////////////////////////////////////////////////////
0:             parameters.getPartitionID(), parameters.getSegmentId(), false, false);
author:jackylk
-------------------------------------------------------------------------------
commit:dc83b2a
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1: import org.apache.carbondata.core.datastore.row.CarbonRow;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:     ExecutorService executorService = Executors.newFixedThreadPool(iterators.length);
1:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
commit:98df130
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getMeasureDataType(), sortParameters.getNoDictionaryDimnesionColumn(),
commit:ce09aaa
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.core.metadata.schema.BucketingInfo;
author:QiangCai
-------------------------------------------------------------------------------
commit:81149f6
/////////////////////////////////////////////////////////////////////////
1:     return new SingleThreadFinalSortFilesMerger(dataFolderLocation, sortParameters.getTableName(),
commit:9f94529
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getAggType(), sortParameters.getNoDictionaryDimnesionColumn(),
0:             this.sortParameters.getNoDictionarySortColumn());
commit:c5aba5f
/////////////////////////////////////////////////////////////////////////
1:           while (batch.hasNext()) {
1:             CarbonRow row = batch.next();
/////////////////////////////////////////////////////////////////////////
1:       CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
commit:256dbed
/////////////////////////////////////////////////////////////////////////
0:     sortBufferSize = buffer / bucketingInfo.getNumberOfBuckets();
commit:41347d8
/////////////////////////////////////////////////////////////////////////
1:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:  * contributor license agreements.  See the NOTICE file distributed with
1:  * this work for additional information regarding copyright ownership.
1:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:  * (the "License"); you may not use this file except in compliance with
1:  * the License.  You may obtain a copy of the License at
1:  * Unless required by applicable law or agreed to in writing, software
1:  * distributed under the License is distributed on an "AS IS" BASIS,
1:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:  * See the License for the specific language governing permissions and
1:  * limitations under the License.
author:ravipesala
-------------------------------------------------------------------------------
commit:f82b10b
/////////////////////////////////////////////////////////////////////////
1:   private SortIntermediateFileMerger[] intermediateFileMergers;
1: 
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0:     intermediateFileMergers =
0:         new SortIntermediateFileMerger[sortDataRows.length];
1:         intermediateFileMergers[i] = new SortIntermediateFileMerger(parameters);
1:         sortDataRows[i] = new SortDataRows(parameters, intermediateFileMergers[i]);
/////////////////////////////////////////////////////////////////////////
1:       for (int i = 0; i < intermediateFileMergers.length; i++) {
1:         intermediateFileMergers[i].finish();
1:       }
/////////////////////////////////////////////////////////////////////////
1:     for (int i = 0; i < intermediateFileMergers.length; i++) {
1:       intermediateFileMergers[i].close();
1:     }
commit:e6b6090
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getAggType(), sortParameters.getNoDictionaryDimnesionColumn());
commit:30f575f
/////////////////////////////////////////////////////////////////////////
1: import java.util.concurrent.atomic.AtomicLong;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:   private AtomicLong rowCounter;
1: 
0:   public ParallelReadMergeSorterWithBucketingImpl(AtomicLong rowCounter,
1:     this.rowCounter = rowCounter;
/////////////////////////////////////////////////////////////////////////
0:         executorService.submit(new SortIteratorThread(iterators[i], sortDataRows, rowCounter));
/////////////////////////////////////////////////////////////////////////
1:     private AtomicLong rowCounter;
1: 
1:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortDataRows[] sortDataRows,
0:         AtomicLong rowCounter) {
1:       this.rowCounter = rowCounter;
/////////////////////////////////////////////////////////////////////////
1:                 rowCounter.getAndAdd(1);
commit:cbf8797
/////////////////////////////////////////////////////////////////////////
1: /*
0:  * Licensed to the Apache Software Foundation (ASF) under one
0:  * or more contributor license agreements.  See the NOTICE file
0:  * distributed with this work for additional information
0:  * regarding copyright ownership.  The ASF licenses this file
0:  * to you under the Apache License, Version 2.0 (the
0:  * "License"); you may not use this file except in compliance
0:  * with the License.  You may obtain a copy of the License at
1:  *
1:  *    http://www.apache.org/licenses/LICENSE-2.0
1:  *
0:  * Unless required by applicable law or agreed to in writing,
0:  * software distributed under the License is distributed on an
0:  * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
0:  * KIND, either express or implied.  See the License for the
0:  * specific language governing permissions and limitations
0:  * under the License.
1:  */
0: package org.apache.carbondata.processing.newflow.sort.impl;
1: 
1: import java.io.File;
1: import java.util.Iterator;
0: import java.util.concurrent.Callable;
1: import java.util.concurrent.ExecutorService;
1: import java.util.concurrent.Executors;
1: import java.util.concurrent.TimeUnit;
1: 
1: import org.apache.carbondata.common.CarbonIterator;
1: import org.apache.carbondata.common.logging.LogService;
1: import org.apache.carbondata.common.logging.LogServiceFactory;
0: import org.apache.carbondata.core.carbon.metadata.schema.BucketingInfo;
1: import org.apache.carbondata.core.constants.CarbonCommonConstants;
1: import org.apache.carbondata.core.util.CarbonProperties;
1: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
0: import org.apache.carbondata.processing.newflow.DataField;
0: import org.apache.carbondata.processing.newflow.exception.CarbonDataLoadingException;
0: import org.apache.carbondata.processing.newflow.row.CarbonRow;
0: import org.apache.carbondata.processing.newflow.row.CarbonRowBatch;
0: import org.apache.carbondata.processing.newflow.sort.Sorter;
0: import org.apache.carbondata.processing.sortandgroupby.exception.CarbonSortKeyAndGroupByException;
0: import org.apache.carbondata.processing.sortandgroupby.sortdata.SortDataRows;
0: import org.apache.carbondata.processing.sortandgroupby.sortdata.SortIntermediateFileMerger;
0: import org.apache.carbondata.processing.sortandgroupby.sortdata.SortParameters;
0: import org.apache.carbondata.processing.store.SingleThreadFinalSortFilesMerger;
0: import org.apache.carbondata.processing.store.writer.exception.CarbonDataWriterException;
1: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1: 
1: /**
1:  * It parallely reads data from array of iterates and do merge sort.
1:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:  * final merge sort result.
0:  * This step is specifically for bucketing, it sorts each bucket data separately and write to
0:  * temp files.
1:  */
0: public class ParallelReadMergeSorterWithBucketingImpl implements Sorter {
1: 
0:   private static final LogService LOGGER =
0:       LogServiceFactory.getLogService(ParallelReadMergeSorterImpl.class.getName());
1: 
0:   private SortParameters sortParameters;
1: 
0:   private SortIntermediateFileMerger intermediateFileMerger;
1: 
0:   private ExecutorService executorService;
1: 
0:   private BucketingInfo bucketingInfo;
1: 
0:   private DataField[] inputDataFields;
1: 
1:   private int sortBufferSize;
1: 
0:   public ParallelReadMergeSorterWithBucketingImpl(DataField[] inputDataFields,
0:       BucketingInfo bucketingInfo) {
0:     this.inputDataFields = inputDataFields;
0:     this.bucketingInfo = bucketingInfo;
1:   }
1: 
0:   @Override public void initialize(SortParameters sortParameters) {
0:     this.sortParameters = sortParameters;
0:     intermediateFileMerger = new SortIntermediateFileMerger(sortParameters);
1:     int buffer = Integer.parseInt(CarbonProperties.getInstance()
1:         .getProperty(CarbonCommonConstants.SORT_SIZE, CarbonCommonConstants.SORT_SIZE_DEFAULT_VAL));
0:     sortBufferSize = buffer/bucketingInfo.getNumberOfBuckets();
1:     if (sortBufferSize < 100) {
1:       sortBufferSize = 100;
1:     }
1:   }
1: 
1:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:       throws CarbonDataLoadingException {
0:     SortDataRows[] sortDataRows = new SortDataRows[bucketingInfo.getNumberOfBuckets()];
1:     try {
0:       for (int i = 0; i < bucketingInfo.getNumberOfBuckets(); i++) {
0:         SortParameters parameters = sortParameters.getCopy();
1:         parameters.setPartitionID(i + "");
1:         setTempLocation(parameters);
1:         parameters.setBufferSize(sortBufferSize);
0:         sortDataRows[i] = new SortDataRows(parameters, intermediateFileMerger);
1:         sortDataRows[i].initialize();
1:       }
1:     } catch (CarbonSortKeyAndGroupByException e) {
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
0:       intermediateFileMerger.finish();
1:     } catch (CarbonDataWriterException e) {
1:       throw new CarbonDataLoadingException(e);
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1: 
0:     Iterator<CarbonRowBatch>[] batchIterator = new Iterator[bucketingInfo.getNumberOfBuckets()];
0:     for (int i = 0; i < bucketingInfo.getNumberOfBuckets(); i++) {
0:       batchIterator[i] = new MergedDataIterator(String.valueOf(i), batchSize);
1:     }
1: 
1:     return batchIterator;
1:   }
1: 
0:   private SingleThreadFinalSortFilesMerger getFinalMerger(String bucketId) {
0:     String storeLocation = CarbonDataProcessorUtil
0:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
0:             String.valueOf(sortParameters.getTaskNo()), bucketId,
0:             sortParameters.getSegmentId() + "", false);
1:     // Set the data file location
0:     String dataFolderLocation =
0:         storeLocation + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION;
0:     SingleThreadFinalSortFilesMerger finalMerger =
0:         new SingleThreadFinalSortFilesMerger(dataFolderLocation, sortParameters.getTableName(),
0:             sortParameters.getDimColCount(), sortParameters.getComplexDimColCount(),
0:             sortParameters.getMeasureColCount(), sortParameters.getNoDictionaryCount(),
0:             sortParameters.getAggType(), sortParameters.getNoDictionaryDimnesionColumn(),
0:             sortParameters.isUseKettle());
0:     return finalMerger;
1:   }
1: 
1:   @Override public void close() {
0:     intermediateFileMerger.close();
1:   }
1: 
1:   /**
1:    * Below method will be used to process data to next step
1:    */
1:   private boolean processRowToNextStep(SortDataRows[] sortDataRows, SortParameters parameters)
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
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1:   }
1: 
1:   private void setTempLocation(SortParameters parameters) {
0:     String carbonDataDirectoryPath = CarbonDataProcessorUtil
0:         .getLocalDataFolderLocation(parameters.getDatabaseName(),
0:             parameters.getTableName(), parameters.getTaskNo(),
0:             parameters.getPartitionID(), parameters.getSegmentId(), false);
0:     parameters.setTempFileLocation(
0:         carbonDataDirectoryPath + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:   }
1: 
1:   /**
1:    * This thread iterates the iterator and adds the rows to @{@link SortDataRows}
1:    */
0:   private static class SortIteratorThread implements Callable<Void> {
1: 
1:     private Iterator<CarbonRowBatch> iterator;
1: 
1:     private SortDataRows[] sortDataRows;
1: 
0:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortDataRows[] sortDataRows) {
1:       this.iterator = iterator;
1:       this.sortDataRows = sortDataRows;
1:     }
1: 
0:     @Override public Void call() throws CarbonDataLoadingException {
1:       try {
1:         while (iterator.hasNext()) {
1:           CarbonRowBatch batch = iterator.next();
0:           Iterator<CarbonRow> batchIterator = batch.getBatchIterator();
0:           int i = 0;
0:           while (batchIterator.hasNext()) {
0:             CarbonRow row = batchIterator.next();
1:             if (row != null) {
0:               SortDataRows sortDataRow = sortDataRows[row.bucketNumber];
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
1:     private boolean firstRow = true;
1: 
0:     public MergedDataIterator(String partitionId, int batchSize) {
0:       this.partitionId = partitionId;
1:       this.batchSize = batchSize;
1:     }
1: 
1:     private SingleThreadFinalSortFilesMerger finalMerger;
1: 
1:     @Override public boolean hasNext() {
1:       if (firstRow) {
1:         firstRow = false;
0:         finalMerger = getFinalMerger(partitionId);
1:         finalMerger.startFinalMerge();
1:       }
1:       return finalMerger.hasNext();
1:     }
1: 
1:     @Override public CarbonRowBatch next() {
1:       int counter = 0;
0:       CarbonRowBatch rowBatch = new CarbonRowBatch();
1:       while (finalMerger.hasNext() && counter < batchSize) {
1:         rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:         counter++;
1:       }
1:       return rowBatch;
1:     }
1:   }
1: }
author:mohammadshahidkhan
-------------------------------------------------------------------------------
commit:53accb3
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.processing.newflow.sort.AbstractMergeSorter;
/////////////////////////////////////////////////////////////////////////
0: public class ParallelReadMergeSorterWithBucketingImpl extends AbstractMergeSorter {
/////////////////////////////////////////////////////////////////////////
0:     this.threadStatusObserver = new ThreadStatusObserver(this.executorService);
0:         executorService.submit(new SortIteratorThread(iterators[i], sortDataRows, rowCounter,
0:             this.threadStatusObserver));
1:       checkError();
1:     checkError();
/////////////////////////////////////////////////////////////////////////
1:     private ThreadStatusObserver threadStatusObserver;
0: 
0:         AtomicLong rowCounter, ThreadStatusObserver observer) {
1:       this.threadStatusObserver = observer;
/////////////////////////////////////////////////////////////////////////
1:         this.threadStatusObserver.notifyFailed(e);
============================================================================
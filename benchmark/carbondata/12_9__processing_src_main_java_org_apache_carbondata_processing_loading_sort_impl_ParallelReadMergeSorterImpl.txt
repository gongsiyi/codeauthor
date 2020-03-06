1:9aee980: /*
1:41347d8:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:41347d8:  * contributor license agreements.  See the NOTICE file distributed with
1:41347d8:  * this work for additional information regarding copyright ownership.
1:41347d8:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:41347d8:  * (the "License"); you may not use this file except in compliance with
1:41347d8:  * the License.  You may obtain a copy of the License at
1:9aee980:  *
1:9aee980:  *    http://www.apache.org/licenses/LICENSE-2.0
1:9aee980:  *
1:41347d8:  * Unless required by applicable law or agreed to in writing, software
1:41347d8:  * distributed under the License is distributed on an "AS IS" BASIS,
1:41347d8:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:41347d8:  * See the License for the specific language governing permissions and
1:41347d8:  * limitations under the License.
2:9aee980:  */
1:349c59c: package org.apache.carbondata.processing.loading.sort.impl;
9:9aee980: 
1:9aee980: import java.io.File;
1:9aee980: import java.util.Iterator;
1:9aee980: import java.util.concurrent.ExecutorService;
1:9aee980: import java.util.concurrent.Executors;
1:9aee980: import java.util.concurrent.TimeUnit;
1:30f575f: import java.util.concurrent.atomic.AtomicLong;
1:9aee980: 
1:9aee980: import org.apache.carbondata.common.CarbonIterator;
1:9aee980: import org.apache.carbondata.common.logging.LogService;
1:9aee980: import org.apache.carbondata.common.logging.LogServiceFactory;
1:9aee980: import org.apache.carbondata.core.constants.CarbonCommonConstants;
1:dc83b2a: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1:dc83b2a: import org.apache.carbondata.core.datastore.row.CarbonRow;
1:63434fa: import org.apache.carbondata.core.util.CarbonProperties;
1:a734add: import org.apache.carbondata.core.util.CarbonThreadFactory;
1:9aee980: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
1:349c59c: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1:349c59c: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1:349c59c: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1:349c59c: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SingleThreadFinalSortFilesMerger;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortDataRows;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortIntermediateFileMerger;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
1:9aee980: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1:9aee980: 
2:9aee980: /**
1:9aee980:  * It parallely reads data from array of iterates and do merge sort.
1:9aee980:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:9aee980:  * final merge sort result.
1:9aee980:  */
1:53accb3: public class ParallelReadMergeSorterImpl extends AbstractMergeSorter {
1:9aee980: 
1:9aee980:   private static final LogService LOGGER =
1:9aee980:       LogServiceFactory.getLogService(ParallelReadMergeSorterImpl.class.getName());
1:9aee980: 
1:9aee980:   private SortParameters sortParameters;
1:9aee980: 
1:9aee980:   private SortIntermediateFileMerger intermediateFileMerger;
1:9aee980: 
1:9aee980:   private SingleThreadFinalSortFilesMerger finalMerger;
1:9aee980: 
1:30f575f:   private AtomicLong rowCounter;
1:30f575f: 
1:a734add:   private ExecutorService executorService;
1:a734add: 
1:30f575f:   public ParallelReadMergeSorterImpl(AtomicLong rowCounter) {
1:30f575f:     this.rowCounter = rowCounter;
8:9aee980:   }
1:9aee980: 
1:9aee980:   @Override
1:9aee980:   public void initialize(SortParameters sortParameters) {
1:9aee980:     this.sortParameters = sortParameters;
1:9aee980:     intermediateFileMerger = new SortIntermediateFileMerger(sortParameters);
1:ded8b41:     String[] storeLocations =
1:9aee980:         CarbonDataProcessorUtil.getLocalDataFolderLocation(
1:9aee980:             sortParameters.getDatabaseName(), sortParameters.getTableName(),
1:5bedd77:             String.valueOf(sortParameters.getTaskNo()), sortParameters.getSegmentId(),
1:5bedd77:             false, false);
1:9aee980:     // Set the data file location
1:ded8b41:     String[] dataFolderLocations = CarbonDataProcessorUtil.arrayAppend(storeLocations,
1:ded8b41:         File.separator, CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:9aee980:     finalMerger =
1:ded8b41:         new SingleThreadFinalSortFilesMerger(dataFolderLocations, sortParameters.getTableName(),
1:c100251:             sortParameters);
1:9aee980:   }
1:9aee980: 
1:9aee980:   @Override
1:9aee980:   public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
2:9aee980:       throws CarbonDataLoadingException {
1:63434fa:     SortDataRows sortDataRow = new SortDataRows(sortParameters, intermediateFileMerger);
1:63434fa:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
2:9aee980:     try {
1:63434fa:       sortDataRow.initialize();
3:9aee980:     } catch (CarbonSortKeyAndGroupByException e) {
3:9aee980:       throw new CarbonDataLoadingException(e);
1:63434fa:     }
1:a734add:     this.executorService = Executors.newFixedThreadPool(iterators.length,
1:a734add:         new CarbonThreadFactory("SafeParallelSorterPool:" + sortParameters.getTableName()));
1:9e11e13:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
1:63434fa: 
1:9aee980:     try {
1:63434fa:       for (int i = 0; i < iterators.length; i++) {
1:06b0d08:         executorService.execute(
1:9e11e13:             new SortIteratorThread(iterators[i], sortDataRow, batchSize, rowCounter,
1:9e11e13:                 threadStatusObserver));
1:496cde4:       }
1:9aee980:       executorService.shutdown();
1:9aee980:       executorService.awaitTermination(2, TimeUnit.DAYS);
1:63434fa:       processRowToNextStep(sortDataRow, sortParameters);
1:496cde4:     } catch (Exception e) {
1:9e11e13:       checkError();
1:9aee980:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:9aee980:     }
1:9e11e13:     checkError();
1:9aee980:     try {
1:9aee980:       intermediateFileMerger.finish();
1:c5aba5f:       intermediateFileMerger = null;
1:9aee980:       finalMerger.startFinalMerge();
1:9aee980:     } catch (CarbonDataWriterException e) {
1:9aee980:       throw new CarbonDataLoadingException(e);
1:9aee980:     } catch (CarbonSortKeyAndGroupByException e) {
1:9aee980:       throw new CarbonDataLoadingException(e);
1:9aee980:     }
1:9aee980: 
1:9aee980:     // Creates the iterator to read from merge sorter.
1:9aee980:     Iterator<CarbonRowBatch> batchIterator = new CarbonIterator<CarbonRowBatch>() {
1:9aee980: 
1:9aee980:       @Override
1:9aee980:       public boolean hasNext() {
1:9aee980:         return finalMerger.hasNext();
1:9aee980:       }
1:9aee980: 
1:9aee980:       @Override
1:9aee980:       public CarbonRowBatch next() {
1:9aee980:         int counter = 0;
1:c5aba5f:         CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
1:9aee980:         while (finalMerger.hasNext() && counter < batchSize) {
1:9aee980:           rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:9aee980:           counter++;
1:9aee980:         }
1:9aee980:         return rowBatch;
1:9aee980:       }
1:9aee980:     };
1:9aee980:     return new Iterator[] { batchIterator };
1:9aee980:   }
1:9aee980: 
1:9aee980:   @Override public void close() {
1:c5aba5f:     if (intermediateFileMerger != null) {
1:c5aba5f:       intermediateFileMerger.close();
1:c5aba5f:     }
1:a734add:     if (null != executorService && !executorService.isShutdown()) {
1:a734add:       executorService.shutdownNow();
1:a734add:     }
1:9aee980:   }
1:9aee980: 
1:9aee980:   /**
1:63434fa:    * Below method will be used to process data to next step
1:63434fa:    */
1:63434fa:   private boolean processRowToNextStep(SortDataRows sortDataRows, SortParameters parameters)
1:63434fa:       throws CarbonDataLoadingException {
1:63434fa:     try {
1:63434fa:       // start sorting
1:63434fa:       sortDataRows.startSorting();
1:63434fa: 
1:63434fa:       // check any more rows are present
2:63434fa:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:63434fa:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:63434fa:           .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:63434fa:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:63434fa:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(),
1:63434fa:               System.currentTimeMillis());
2:63434fa:       return false;
1:63434fa:     } catch (CarbonSortKeyAndGroupByException e) {
1:63434fa:       throw new CarbonDataLoadingException(e);
1:63434fa:     }
1:63434fa:   }
1:63434fa: 
1:63434fa:   /**
1:9aee980:    * This thread iterates the iterator and adds the rows to @{@link SortDataRows}
1:9aee980:    */
1:06b0d08:   private static class SortIteratorThread implements Runnable {
1:9aee980: 
1:9aee980:     private Iterator<CarbonRowBatch> iterator;
1:9aee980: 
1:9aee980:     private SortDataRows sortDataRows;
1:9aee980: 
1:63434fa:     private Object[][] buffer;
1:30f575f: 
1:30f575f:     private AtomicLong rowCounter;
1:63434fa: 
1:9e11e13:     private ThreadStatusObserver observer;
1:9aee980: 
1:9aee980:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortDataRows sortDataRows,
1:9e11e13:         int batchSize, AtomicLong rowCounter, ThreadStatusObserver observer) {
1:9aee980:       this.iterator = iterator;
1:9aee980:       this.sortDataRows = sortDataRows;
1:63434fa:       this.buffer = new Object[batchSize][];
1:30f575f:       this.rowCounter = rowCounter;
1:9e11e13:       this.observer = observer;
1:9aee980: 
1:9aee980:     }
1:9aee980: 
1:9aee980:     @Override
1:06b0d08:     public void run() {
1:9aee980:       try {
1:9aee980:         while (iterator.hasNext()) {
1:9aee980:           CarbonRowBatch batch = iterator.next();
1:63434fa:           int i = 0;
1:c5aba5f:           while (batch.hasNext()) {
1:c5aba5f:             CarbonRow row = batch.next();
1:496cde4:             if (row != null) {
1:63434fa:               buffer[i++] = row.getData();
1:496cde4:             }
1:9aee980:           }
1:63434fa:           if (i > 0) {
1:63434fa:             sortDataRows.addRowBatch(buffer, i);
1:30f575f:             rowCounter.getAndAdd(i);
1:63434fa:           }
1:9aee980:         }
1:496cde4:       } catch (Exception e) {
1:9aee980:         LOGGER.error(e);
1:9e11e13:         observer.notifyFailed(e);
1:9aee980:       }
1:9aee980:     }
1:9aee980: 
1:9aee980:   }
1:9aee980: }
============================================================================
author:sraghunandan
-------------------------------------------------------------------------------
commit:f911403
/////////////////////////////////////////////////////////////////////////
author:Jacky Li
-------------------------------------------------------------------------------
commit:5bedd77
/////////////////////////////////////////////////////////////////////////
1:             String.valueOf(sortParameters.getTaskNo()), sortParameters.getSegmentId(),
1:             false, false);
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
author:xuchuanyin
-------------------------------------------------------------------------------
commit:c100251
/////////////////////////////////////////////////////////////////////////
1:             sortParameters);
commit:ded8b41
/////////////////////////////////////////////////////////////////////////
1:     String[] storeLocations =
1:     String[] dataFolderLocations = CarbonDataProcessorUtil.arrayAppend(storeLocations,
1:         File.separator, CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:         new SingleThreadFinalSortFilesMerger(dataFolderLocations, sortParameters.getTableName(),
author:kumarvishal
-------------------------------------------------------------------------------
commit:a734add
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.util.CarbonThreadFactory;
/////////////////////////////////////////////////////////////////////////
1:   private ExecutorService executorService;
1: 
/////////////////////////////////////////////////////////////////////////
1:     this.executorService = Executors.newFixedThreadPool(iterators.length,
1:         new CarbonThreadFactory("SafeParallelSorterPool:" + sortParameters.getTableName()));
/////////////////////////////////////////////////////////////////////////
1:     if (null != executorService && !executorService.isShutdown()) {
1:       executorService.shutdownNow();
1:     }
author:Raghunandan S
-------------------------------------------------------------------------------
commit:06b0d08
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:         executorService.execute(
/////////////////////////////////////////////////////////////////////////
1:   private static class SortIteratorThread implements Runnable {
/////////////////////////////////////////////////////////////////////////
1:     public void run() {
/////////////////////////////////////////////////////////////////////////
author:lionelcao
-------------------------------------------------------------------------------
commit:874764f
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getSegmentId() + "", false, false);
author:jackylk
-------------------------------------------------------------------------------
commit:dc83b2a
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1: import org.apache.carbondata.core.datastore.row.CarbonRow;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0:     ExecutorService executorService = Executors.newFixedThreadPool(iterators.length);
commit:98df130
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getNoDictionaryCount(), sortParameters.getMeasureDataType(),
commit:3fe6903
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
author:QiangCai
-------------------------------------------------------------------------------
commit:9f94529
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getNoDictionaryDimnesionColumn(),
0:             sortParameters.getNoDictionarySortColumn());
commit:c5aba5f
/////////////////////////////////////////////////////////////////////////
1:       intermediateFileMerger = null;
/////////////////////////////////////////////////////////////////////////
1:         CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
/////////////////////////////////////////////////////////////////////////
1:     if (intermediateFileMerger != null) {
1:       intermediateFileMerger.close();
1:     }
/////////////////////////////////////////////////////////////////////////
1:           while (batch.hasNext()) {
1:             CarbonRow row = batch.next();
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
author:mohammadshahidkhan
-------------------------------------------------------------------------------
commit:53accb3
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.processing.newflow.sort.AbstractMergeSorter;
/////////////////////////////////////////////////////////////////////////
1: public class ParallelReadMergeSorterImpl extends AbstractMergeSorter {
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
author:ravipesala
-------------------------------------------------------------------------------
commit:e6b6090
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getNoDictionaryDimnesionColumn());
commit:30f575f
/////////////////////////////////////////////////////////////////////////
1: import java.util.concurrent.atomic.AtomicLong;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:   private AtomicLong rowCounter;
1: 
1:   public ParallelReadMergeSorterImpl(AtomicLong rowCounter) {
1:     this.rowCounter = rowCounter;
/////////////////////////////////////////////////////////////////////////
0:             new SortIteratorThread(iterators[i], sortDataRow, batchSize, rowCounter));
/////////////////////////////////////////////////////////////////////////
1:     private AtomicLong rowCounter;
1: 
0:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator,
0:         SortDataRows sortDataRows, int batchSize, AtomicLong rowCounter) {
1:       this.rowCounter = rowCounter;
/////////////////////////////////////////////////////////////////////////
1:             rowCounter.getAndAdd(i);
commit:8100d94
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
commit:63434fa
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.util.CarbonProperties;
/////////////////////////////////////////////////////////////////////////
1:     SortDataRows sortDataRow = new SortDataRows(sortParameters, intermediateFileMerger);
1:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
1:       sortDataRow.initialize();
1:       for (int i = 0; i < iterators.length; i++) {
0:             new SortIteratorThread(iterators[i], sortDataRow, sortParameters, batchSize));
1:       processRowToNextStep(sortDataRow, sortParameters);
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:    * Below method will be used to process data to next step
1:    */
1:   private boolean processRowToNextStep(SortDataRows sortDataRows, SortParameters parameters)
1:       throws CarbonDataLoadingException {
0:     if (null == sortDataRows) {
1:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
0:       LOGGER.info("Number of Records was Zero");
0:       String logMessage = "Summary: Carbon Sort Key Step: Read: " + 0 + ": Write: " + 0;
0:       LOGGER.info(logMessage);
1:       return false;
1:     }
1: 
1:     try {
1:       // start sorting
1:       sortDataRows.startSorting();
1: 
1:       // check any more rows are present
1:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:           .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(),
1:               System.currentTimeMillis());
1:       return false;
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1:   }
1: 
1:   /**
/////////////////////////////////////////////////////////////////////////
1:     private Object[][] buffer;
1: 
0:         SortParameters parameters, int batchSize) {
1:       this.buffer = new Object[batchSize][];
/////////////////////////////////////////////////////////////////////////
1:           int i = 0;
1:               buffer[i++] = row.getData();
1:           if (i > 0) {
1:             sortDataRows.addRowBatch(buffer, i);
1:           }
/////////////////////////////////////////////////////////////////////////
commit:496cde4
/////////////////////////////////////////////////////////////////////////
0:             sortParameters.getDimColCount(),
0:             sortParameters.getNoDictionaryDimnesionColumn(), sortParameters.isUseKettle());
/////////////////////////////////////////////////////////////////////////
0:       for (int i = 0; i < sortDataRows.length; i++) {
0:         executorService.submit(
0:             new SortIteratorThread(iterators[i], sortDataRows[i], sortParameters));
1:       }
1:     } catch (Exception e) {
/////////////////////////////////////////////////////////////////////////
0:             CarbonRow row = batchIterator.next();
1:             if (row != null) {
0:               sortDataRows.addRow(row.getData());
1:             }
1:       } catch (Exception e) {
commit:9aee980
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
1: import org.apache.carbondata.core.constants.CarbonCommonConstants;
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
1:  */
0: public class ParallelReadMergeSorterImpl implements Sorter {
1: 
1:   private static final LogService LOGGER =
1:       LogServiceFactory.getLogService(ParallelReadMergeSorterImpl.class.getName());
1: 
1:   private SortParameters sortParameters;
1: 
1:   private SortIntermediateFileMerger intermediateFileMerger;
1: 
0:   private ExecutorService executorService;
1: 
1:   private SingleThreadFinalSortFilesMerger finalMerger;
1: 
0:   private DataField[] inputDataFields;
1: 
0:   public ParallelReadMergeSorterImpl(DataField[] inputDataFields) {
0:     this.inputDataFields = inputDataFields;
1:   }
1: 
1:   @Override
1:   public void initialize(SortParameters sortParameters) {
1:     this.sortParameters = sortParameters;
1:     intermediateFileMerger = new SortIntermediateFileMerger(sortParameters);
0:     String storeLocation =
1:         CarbonDataProcessorUtil.getLocalDataFolderLocation(
1:             sortParameters.getDatabaseName(), sortParameters.getTableName(),
0:             String.valueOf(sortParameters.getTaskNo()), sortParameters.getPartitionID(),
0:             sortParameters.getSegmentId() + "", false);
1:     // Set the data file location
0:     String dataFolderLocation =
0:         storeLocation + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION;
1:     finalMerger =
0:         new SingleThreadFinalSortFilesMerger(dataFolderLocation, sortParameters.getTableName(),
0:             sortParameters.getDimColCount() - sortParameters.getComplexDimColCount(),
0:             sortParameters.getComplexDimColCount(), sortParameters.getMeasureColCount(),
0:             sortParameters.getNoDictionaryCount(), sortParameters.getAggType(),
0:             sortParameters.getNoDictionaryDimnesionColumn());
1:   }
1: 
1:   @Override
1:   public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:       throws CarbonDataLoadingException {
0:     SortDataRows[] sortDataRows = new SortDataRows[iterators.length];
1:     try {
0:       for (int i = 0; i < iterators.length; i++) {
0:         sortDataRows[i] = new SortDataRows(sortParameters, intermediateFileMerger);
0:         // initialize sort
0:         sortDataRows[i].initialize();
1:       }
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
0:     this.executorService = Executors.newFixedThreadPool(iterators.length);
1: 
0:     // First prepare the data for sort.
0:     Iterator<CarbonRowBatch>[] sortPrepIterators = new Iterator[iterators.length];
0:     for (int i = 0; i < sortPrepIterators.length; i++) {
0:       sortPrepIterators[i] = new SortPreparatorIterator(iterators[i], inputDataFields);
1:     }
1: 
0:     for (int i = 0; i < sortDataRows.length; i++) {
0:       executorService
0:           .submit(new SortIteratorThread(sortPrepIterators[i], sortDataRows[i], sortParameters));
1:     }
1: 
1:     try {
1:       executorService.shutdown();
1:       executorService.awaitTermination(2, TimeUnit.DAYS);
0:     } catch (InterruptedException e) {
1:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:     }
1:     try {
1:       intermediateFileMerger.finish();
1:       finalMerger.startFinalMerge();
1:     } catch (CarbonDataWriterException e) {
1:       throw new CarbonDataLoadingException(e);
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1: 
0:     //TODO get the batch size from CarbonProperties
0:     final int batchSize = 1000;
1: 
1:     // Creates the iterator to read from merge sorter.
1:     Iterator<CarbonRowBatch> batchIterator = new CarbonIterator<CarbonRowBatch>() {
1: 
1:       @Override
1:       public boolean hasNext() {
1:         return finalMerger.hasNext();
1:       }
1: 
1:       @Override
1:       public CarbonRowBatch next() {
1:         int counter = 0;
0:         CarbonRowBatch rowBatch = new CarbonRowBatch();
1:         while (finalMerger.hasNext() && counter < batchSize) {
1:           rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:           counter++;
1:         }
1:         return rowBatch;
1:       }
1:     };
1:     return new Iterator[] { batchIterator };
1:   }
1: 
1:   @Override public void close() {
0:     intermediateFileMerger.close();
1:   }
1: 
1:   /**
1:    * This thread iterates the iterator and adds the rows to @{@link SortDataRows}
1:    */
0:   private static class SortIteratorThread implements Callable<Void> {
1: 
1:     private Iterator<CarbonRowBatch> iterator;
1: 
1:     private SortDataRows sortDataRows;
1: 
0:     private SortParameters parameters;
1: 
1:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortDataRows sortDataRows,
0:         SortParameters parameters) {
1:       this.iterator = iterator;
1:       this.sortDataRows = sortDataRows;
0:       this.parameters = parameters;
1:     }
1: 
1:     @Override
0:     public Void call() throws CarbonDataLoadingException {
1:       try {
1:         while (iterator.hasNext()) {
1:           CarbonRowBatch batch = iterator.next();
0:           Iterator<CarbonRow> batchIterator = batch.getBatchIterator();
0:           while (batchIterator.hasNext()) {
0:             sortDataRows.addRow(batchIterator.next().getData());
1:           }
1:         }
1: 
0:         processRowToNextStep(sortDataRows);
1:       } catch (CarbonSortKeyAndGroupByException e) {
1:         LOGGER.error(e);
1:         throw new CarbonDataLoadingException(e);
1:       }
0:       return null;
1:     }
1: 
1:     /**
0:      * Below method will be used to process data to next step
1:      */
0:     private boolean processRowToNextStep(SortDataRows sortDataRows)
1:         throws CarbonDataLoadingException {
0:       if (null == sortDataRows) {
0:         LOGGER.info("Record Processed For table: " + parameters.getTableName());
0:         LOGGER.info("Number of Records was Zero");
0:         String logMessage = "Summary: Carbon Sort Key Step: Read: " + 0 + ": Write: " + 0;
0:         LOGGER.info(logMessage);
0:         return false;
1:       }
1: 
1:       try {
0:         // start sorting
0:         sortDataRows.startSorting();
1: 
0:         // check any more rows are present
0:         LOGGER.info("Record Processed For table: " + parameters.getTableName());
0:         CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
0:             .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
0:         CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
0:             .recordDictionaryValuesTotalTime(parameters.getPartitionID(),
0:                 System.currentTimeMillis());
0:         return false;
1:       } catch (CarbonSortKeyAndGroupByException e) {
1:         throw new CarbonDataLoadingException(e);
1:       }
1:     }
1:   }
1: }
author:akash
-------------------------------------------------------------------------------
commit:9e11e13
/////////////////////////////////////////////////////////////////////////
0:   private ThreadStatusObserver threadStatusObserver;
0: 
/////////////////////////////////////////////////////////////////////////
1:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
1:             new SortIteratorThread(iterators[i], sortDataRow, batchSize, rowCounter,
1:                 threadStatusObserver));
1:       checkError();
1:     checkError();
/////////////////////////////////////////////////////////////////////////
0:    * Below method will be used to check error in exception
0:    */
0:   private void checkError() {
0:     if (threadStatusObserver.getThrowable() != null) {
0:       if (threadStatusObserver.getThrowable() instanceof CarbonDataLoadingException) {
0:         throw (CarbonDataLoadingException) threadStatusObserver.getThrowable();
0:       } else {
0:         throw new CarbonDataLoadingException(threadStatusObserver.getThrowable());
0:       }
0:     }
0:   }
0:   /**
/////////////////////////////////////////////////////////////////////////
1:     private ThreadStatusObserver observer;
0: 
0:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortDataRows sortDataRows,
1:         int batchSize, AtomicLong rowCounter, ThreadStatusObserver observer) {
1:       this.observer = observer;
0: 
/////////////////////////////////////////////////////////////////////////
1:         observer.notifyFailed(e);
author:WilliamZhu
-------------------------------------------------------------------------------
commit:6c9194d
/////////////////////////////////////////////////////////////////////////
0:   private static final Object taskContext = CarbonDataProcessorUtil.fetchTaskContext();
0: 
/////////////////////////////////////////////////////////////////////////
0:         CarbonDataProcessorUtil.configureTaskContext(taskContext);
============================================================================
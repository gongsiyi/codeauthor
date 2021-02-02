1:f1f9348: /*
1:41347d8:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:41347d8:  * contributor license agreements.  See the NOTICE file distributed with
1:41347d8:  * this work for additional information regarding copyright ownership.
1:41347d8:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:41347d8:  * (the "License"); you may not use this file except in compliance with
1:41347d8:  * the License.  You may obtain a copy of the License at
1:f1f9348:  *
1:f1f9348:  *    http://www.apache.org/licenses/LICENSE-2.0
1:f1f9348:  *
1:41347d8:  * Unless required by applicable law or agreed to in writing, software
1:41347d8:  * distributed under the License is distributed on an "AS IS" BASIS,
1:41347d8:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:41347d8:  * See the License for the specific language governing permissions and
1:41347d8:  * limitations under the License.
1:f1f9348:  */
1:349c59c: package org.apache.carbondata.processing.loading.sort.impl;
4:f1f9348: 
1:f1f9348: import java.util.Iterator;
1:f1f9348: import java.util.List;
1:f1f9348: import java.util.concurrent.ExecutorService;
1:f1f9348: import java.util.concurrent.Executors;
1:f1f9348: import java.util.concurrent.TimeUnit;
1:30f575f: import java.util.concurrent.atomic.AtomicLong;
1:f1f9348: 
1:f1f9348: import org.apache.carbondata.common.CarbonIterator;
1:f1f9348: import org.apache.carbondata.common.logging.LogService;
1:f1f9348: import org.apache.carbondata.common.logging.LogServiceFactory;
1:dc83b2a: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1:dc83b2a: import org.apache.carbondata.core.datastore.row.CarbonRow;
1:f1f9348: import org.apache.carbondata.core.util.CarbonProperties;
1:a734add: import org.apache.carbondata.core.util.CarbonThreadFactory;
1:f1f9348: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
1:349c59c: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1:349c59c: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1:349c59c: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeCarbonRowPage;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeSortDataRows;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeIntermediateMerger;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
1:349c59c: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
1:f1f9348: 
1:f1f9348: /**
1:f1f9348:  * It parallely reads data from array of iterates and do merge sort.
1:f1f9348:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:f1f9348:  * final merge sort result.
1:f1f9348:  */
1:53accb3: public class UnsafeParallelReadMergeSorterImpl extends AbstractMergeSorter {
1:f1f9348: 
1:f1f9348:   private static final LogService LOGGER =
1:f1f9348:       LogServiceFactory.getLogService(UnsafeParallelReadMergeSorterImpl.class.getName());
1:f1f9348: 
1:f1f9348:   private SortParameters sortParameters;
1:f1f9348: 
1:f1f9348:   private UnsafeIntermediateMerger unsafeIntermediateFileMerger;
1:f1f9348: 
1:f1f9348:   private UnsafeSingleThreadFinalSortFilesMerger finalMerger;
1:f1f9348: 
1:30f575f:   private AtomicLong rowCounter;
1:30f575f: 
1:a734add:   private ExecutorService executorService;
1:a734add: 
1:30f575f:   public UnsafeParallelReadMergeSorterImpl(AtomicLong rowCounter) {
1:30f575f:     this.rowCounter = rowCounter;
2:f1f9348:   }
1:f1f9348: 
1:f1f9348:   @Override public void initialize(SortParameters sortParameters) {
1:f1f9348:     this.sortParameters = sortParameters;
1:f1f9348:     unsafeIntermediateFileMerger = new UnsafeIntermediateMerger(sortParameters);
1:ded8b41: 
1:f82b10b:     finalMerger = new UnsafeSingleThreadFinalSortFilesMerger(sortParameters,
1:f82b10b:         sortParameters.getTempFileLocation());
1:f1f9348:   }
1:f1f9348: 
1:f1f9348:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:f1f9348:       throws CarbonDataLoadingException {
1:f82b10b:     int inMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
1:f1f9348:     UnsafeSortDataRows sortDataRow =
1:f82b10b:         new UnsafeSortDataRows(sortParameters, unsafeIntermediateFileMerger, inMemoryChunkSizeInMB);
1:f1f9348:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
1:f1f9348:     try {
1:f1f9348:       sortDataRow.initialize();
1:b439b00:     } catch (Exception e) {
2:f1f9348:       throw new CarbonDataLoadingException(e);
1:f1f9348:     }
1:a734add:     this.executorService = Executors.newFixedThreadPool(iterators.length,
1:a734add:         new CarbonThreadFactory("UnsafeParallelSorterPool:" + sortParameters.getTableName()));
1:dc83b2a:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
1:f1f9348: 
1:f1f9348:     try {
1:f1f9348:       for (int i = 0; i < iterators.length; i++) {
1:06b0d08:         executorService.execute(
1:53accb3:             new SortIteratorThread(iterators[i], sortDataRow, batchSize, rowCounter,
1:53accb3:                 this.threadStatusObserver));
1:f1f9348:       }
1:f1f9348:       executorService.shutdown();
1:f1f9348:       executorService.awaitTermination(2, TimeUnit.DAYS);
1:3f33276:       if (!sortParameters.getObserver().isFailed()) {
1:3f33276:         processRowToNextStep(sortDataRow, sortParameters);
1:3f33276:       }
1:f1f9348:     } catch (Exception e) {
1:53accb3:       checkError();
1:f1f9348:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:f1f9348:     }
1:53accb3:     checkError();
1:f1f9348:     try {
1:f1f9348:       unsafeIntermediateFileMerger.finish();
1:f1f9348:       List<UnsafeCarbonRowPage> rowPages = unsafeIntermediateFileMerger.getRowPages();
1:f1f9348:       finalMerger.startFinalMerge(rowPages.toArray(new UnsafeCarbonRowPage[rowPages.size()]),
1:f1f9348:           unsafeIntermediateFileMerger.getMergedPages());
1:f1f9348:     } catch (CarbonDataWriterException e) {
1:f1f9348:       throw new CarbonDataLoadingException(e);
3:f1f9348:     } catch (CarbonSortKeyAndGroupByException e) {
1:f1f9348:       throw new CarbonDataLoadingException(e);
1:f1f9348:     }
1:f1f9348: 
1:f1f9348:     // Creates the iterator to read from merge sorter.
1:f1f9348:     Iterator<CarbonRowBatch> batchIterator = new CarbonIterator<CarbonRowBatch>() {
1:f1f9348: 
1:f1f9348:       @Override public boolean hasNext() {
1:f1f9348:         return finalMerger.hasNext();
1:f1f9348:       }
1:f1f9348: 
1:f1f9348:       @Override public CarbonRowBatch next() {
1:f1f9348:         int counter = 0;
1:c5aba5f:         CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
1:f1f9348:         while (finalMerger.hasNext() && counter < batchSize) {
1:f1f9348:           rowBatch.addRow(new CarbonRow(finalMerger.next()));
1:f1f9348:           counter++;
1:f1f9348:         }
1:f1f9348:         return rowBatch;
1:f1f9348:       }
1:f1f9348:     };
1:f1f9348:     return new Iterator[] { batchIterator };
1:f1f9348:   }
1:f1f9348: 
1:f1f9348:   @Override public void close() {
1:a734add:     if (null != executorService && !executorService.isShutdown()) {
1:a734add:       executorService.shutdownNow();
1:a734add:     }
1:f1f9348:     unsafeIntermediateFileMerger.close();
1:f1f9348:     finalMerger.clear();
1:f1f9348:   }
1:f1f9348: 
1:f1f9348:   /**
1:f1f9348:    * Below method will be used to process data to next step
1:f1f9348:    */
1:f1f9348:   private boolean processRowToNextStep(UnsafeSortDataRows sortDataRows, SortParameters parameters)
1:f1f9348:       throws CarbonDataLoadingException {
1:f1f9348:     try {
1:f1f9348:       // start sorting
1:f1f9348:       sortDataRows.startSorting();
1:f1f9348: 
1:f1f9348:       // check any more rows are present
2:f1f9348:       LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:f1f9348:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:f1f9348:           .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:f1f9348:       CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:f1f9348:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
2:f1f9348:       return false;
1:b439b00:     } catch (Exception e) {
1:f1f9348:       throw new CarbonDataLoadingException(e);
1:f1f9348:     }
1:f1f9348:   }
1:f1f9348: 
1:f1f9348:   /**
1:f1f9348:    * This thread iterates the iterator and adds the rows
1:f1f9348:    */
1:06b0d08:   private static class SortIteratorThread implements Runnable {
1:f1f9348: 
1:f1f9348:     private Iterator<CarbonRowBatch> iterator;
1:f1f9348: 
1:f1f9348:     private UnsafeSortDataRows sortDataRows;
1:f1f9348: 
1:f1f9348:     private Object[][] buffer;
1:30f575f: 
1:30f575f:     private AtomicLong rowCounter;
1:f1f9348: 
1:53accb3:     private ThreadStatusObserver threadStatusObserver;
1:f1f9348: 
1:30f575f:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator,
1:53accb3:         UnsafeSortDataRows sortDataRows, int batchSize, AtomicLong rowCounter,
1:53accb3:         ThreadStatusObserver threadStatusObserver) {
1:f1f9348:       this.iterator = iterator;
1:f1f9348:       this.sortDataRows = sortDataRows;
1:f1f9348:       this.buffer = new Object[batchSize][];
1:30f575f:       this.rowCounter = rowCounter;
1:53accb3:       this.threadStatusObserver = threadStatusObserver;
1:f1f9348:     }
1:f1f9348: 
1:06b0d08:     @Override
1:06b0d08:     public void run() {
1:f1f9348:       try {
1:f1f9348:         while (iterator.hasNext()) {
1:f1f9348:           CarbonRowBatch batch = iterator.next();
1:f1f9348:           int i = 0;
1:c5aba5f:           while (batch.hasNext()) {
1:c5aba5f:             CarbonRow row = batch.next();
1:f1f9348:             if (row != null) {
1:f1f9348:               buffer[i++] = row.getData();
1:f1f9348:             }
1:f1f9348:           }
1:f1f9348:           if (i > 0) {
1:f1f9348:             sortDataRows.addRowBatch(buffer, i);
1:30f575f:             rowCounter.getAndAdd(i);
1:f1f9348:           }
1:f1f9348:         }
1:f1f9348:       } catch (Exception e) {
1:f1f9348:         LOGGER.error(e);
1:53accb3:         this.threadStatusObserver.notifyFailed(e);
1:f1f9348:       }
1:f1f9348:     }
1:f1f9348: 
1:f1f9348:   }
1:f1f9348: }
============================================================================
author:sraghunandan
-------------------------------------------------------------------------------
commit:f911403
/////////////////////////////////////////////////////////////////////////
author:BJangir
-------------------------------------------------------------------------------
commit:3f33276
/////////////////////////////////////////////////////////////////////////
1:       if (!sortParameters.getObserver().isFailed()) {
1:         processRowToNextStep(sortDataRow, sortParameters);
1:       }
author:xuchuanyin
-------------------------------------------------------------------------------
commit:b439b00
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:     } catch (Exception e) {
/////////////////////////////////////////////////////////////////////////
1:     } catch (Exception e) {
commit:ded8b41
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1: 
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
1:         new CarbonThreadFactory("UnsafeParallelSorterPool:" + sortParameters.getTableName()));
/////////////////////////////////////////////////////////////////////////
1:     if (null != executorService && !executorService.isShutdown()) {
1:       executorService.shutdownNow();
1:     }
author:Jacky Li
-------------------------------------------------------------------------------
commit:349c59c
/////////////////////////////////////////////////////////////////////////
1: package org.apache.carbondata.processing.loading.sort.impl;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeCarbonRowPage;
1: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeSortDataRows;
1: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeIntermediateMerger;
1: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
1: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
author:Raghunandan S
-------------------------------------------------------------------------------
commit:06b0d08
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:         executorService.execute(
/////////////////////////////////////////////////////////////////////////
1:   private static class SortIteratorThread implements Runnable {
/////////////////////////////////////////////////////////////////////////
1:     @Override
1:     public void run() {
/////////////////////////////////////////////////////////////////////////
author:jackylk
-------------------------------------------------------------------------------
commit:edda248
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.core.memory.MemoryException;
/////////////////////////////////////////////////////////////////////////
0:     } catch (MemoryException e) {
commit:dc83b2a
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1: import org.apache.carbondata.core.datastore.row.CarbonRow;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0:     ExecutorService executorService = Executors.newFixedThreadPool(iterators.length);
1:     this.threadStatusObserver = new ThreadStatusObserver(executorService);
commit:eaadc88
/////////////////////////////////////////////////////////////////////////
0:     } catch (InterruptedException e) {
commit:3fe6903
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
author:ravipesala
-------------------------------------------------------------------------------
commit:f82b10b
/////////////////////////////////////////////////////////////////////////
1:     finalMerger = new UnsafeSingleThreadFinalSortFilesMerger(sortParameters,
1:         sortParameters.getTempFileLocation());
1:     int inMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
1:         new UnsafeSortDataRows(sortParameters, unsafeIntermediateFileMerger, inMemoryChunkSizeInMB);
commit:30f575f
/////////////////////////////////////////////////////////////////////////
1: import java.util.concurrent.atomic.AtomicLong;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:   private AtomicLong rowCounter;
1: 
1:   public UnsafeParallelReadMergeSorterImpl(AtomicLong rowCounter) {
1:     this.rowCounter = rowCounter;
/////////////////////////////////////////////////////////////////////////
0:             .submit(new SortIteratorThread(iterators[i], sortDataRow, batchSize, rowCounter));
/////////////////////////////////////////////////////////////////////////
1:     private AtomicLong rowCounter;
1: 
1:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator,
0:         UnsafeSortDataRows sortDataRows, int batchSize, AtomicLong rowCounter) {
1:       this.rowCounter = rowCounter;
/////////////////////////////////////////////////////////////////////////
1:             rowCounter.getAndAdd(i);
commit:f1f9348
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
0: import java.io.File;
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
0: import org.apache.carbondata.core.constants.CarbonCommonConstants;
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
0: import org.apache.carbondata.processing.store.writer.exception.CarbonDataWriterException;
0: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1: 
1: /**
1:  * It parallely reads data from array of iterates and do merge sort.
1:  * First it sorts the data and write to temp files. These temp files will be merge sorted to get
1:  * final merge sort result.
1:  */
0: public class UnsafeParallelReadMergeSorterImpl implements Sorter {
1: 
1:   private static final LogService LOGGER =
1:       LogServiceFactory.getLogService(UnsafeParallelReadMergeSorterImpl.class.getName());
1: 
1:   private SortParameters sortParameters;
1: 
1:   private UnsafeIntermediateMerger unsafeIntermediateFileMerger;
1: 
0:   private ExecutorService executorService;
1: 
1:   private UnsafeSingleThreadFinalSortFilesMerger finalMerger;
1: 
0:   private DataField[] inputDataFields;
1: 
0:   public UnsafeParallelReadMergeSorterImpl(DataField[] inputDataFields) {
0:     this.inputDataFields = inputDataFields;
1:   }
1: 
1:   @Override public void initialize(SortParameters sortParameters) {
1:     this.sortParameters = sortParameters;
1:     unsafeIntermediateFileMerger = new UnsafeIntermediateMerger(sortParameters);
0:     String storeLocation = CarbonDataProcessorUtil
0:         .getLocalDataFolderLocation(sortParameters.getDatabaseName(), sortParameters.getTableName(),
0:             String.valueOf(sortParameters.getTaskNo()), sortParameters.getPartitionID(),
0:             sortParameters.getSegmentId() + "", false);
0:     // Set the data file location
0:     String dataFolderLocation =
0:         storeLocation + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION;
0:     finalMerger = new UnsafeSingleThreadFinalSortFilesMerger(sortParameters);
1:   }
1: 
1:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:       throws CarbonDataLoadingException {
1:     UnsafeSortDataRows sortDataRow =
0:         new UnsafeSortDataRows(sortParameters, unsafeIntermediateFileMerger);
1:     final int batchSize = CarbonProperties.getInstance().getBatchSize();
1:     try {
1:       sortDataRow.initialize();
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
0:     this.executorService = Executors.newFixedThreadPool(iterators.length);
1: 
1:     try {
1:       for (int i = 0; i < iterators.length; i++) {
0:         executorService
0:             .submit(new SortIteratorThread(iterators[i], sortDataRow, sortParameters, batchSize));
1:       }
1:       executorService.shutdown();
1:       executorService.awaitTermination(2, TimeUnit.DAYS);
0:       processRowToNextStep(sortDataRow, sortParameters);
1:     } catch (Exception e) {
1:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:     }
1:     try {
1:       unsafeIntermediateFileMerger.finish();
1:       List<UnsafeCarbonRowPage> rowPages = unsafeIntermediateFileMerger.getRowPages();
1:       finalMerger.startFinalMerge(rowPages.toArray(new UnsafeCarbonRowPage[rowPages.size()]),
1:           unsafeIntermediateFileMerger.getMergedPages());
1:     } catch (CarbonDataWriterException e) {
1:       throw new CarbonDataLoadingException(e);
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1: 
1:     // Creates the iterator to read from merge sorter.
1:     Iterator<CarbonRowBatch> batchIterator = new CarbonIterator<CarbonRowBatch>() {
1: 
1:       @Override public boolean hasNext() {
1:         return finalMerger.hasNext();
1:       }
1: 
1:       @Override public CarbonRowBatch next() {
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
1:     unsafeIntermediateFileMerger.close();
1:     finalMerger.clear();
1:   }
1: 
1:   /**
1:    * Below method will be used to process data to next step
1:    */
1:   private boolean processRowToNextStep(UnsafeSortDataRows sortDataRows, SortParameters parameters)
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
1:           .recordDictionaryValuesTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:       return false;
1:     } catch (CarbonSortKeyAndGroupByException e) {
1:       throw new CarbonDataLoadingException(e);
1:     }
1:   }
1: 
1:   /**
1:    * This thread iterates the iterator and adds the rows
1:    */
0:   private static class SortIteratorThread implements Callable<Void> {
1: 
1:     private Iterator<CarbonRowBatch> iterator;
1: 
1:     private UnsafeSortDataRows sortDataRows;
1: 
0:     private SortParameters parameters;
1: 
1:     private Object[][] buffer;
1: 
0:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, UnsafeSortDataRows sortDataRows,
0:         SortParameters parameters, int batchSize) {
1:       this.iterator = iterator;
1:       this.sortDataRows = sortDataRows;
0:       this.parameters = parameters;
1:       this.buffer = new Object[batchSize][];
1:     }
1: 
0:     @Override public Void call() throws CarbonDataLoadingException {
1:       try {
1:         while (iterator.hasNext()) {
1:           CarbonRowBatch batch = iterator.next();
0:           Iterator<CarbonRow> batchIterator = batch.getBatchIterator();
1:           int i = 0;
0:           while (batchIterator.hasNext()) {
0:             CarbonRow row = batchIterator.next();
1:             if (row != null) {
1:               buffer[i++] = row.getData();
1:             }
1:           }
1:           if (i > 0) {
1:             sortDataRows.addRowBatch(buffer, i);
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
1: }
author:mohammadshahidkhan
-------------------------------------------------------------------------------
commit:53accb3
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.processing.newflow.sort.AbstractMergeSorter;
/////////////////////////////////////////////////////////////////////////
1: public class UnsafeParallelReadMergeSorterImpl extends AbstractMergeSorter {
/////////////////////////////////////////////////////////////////////////
0:     this.threadStatusObserver = new ThreadStatusObserver(this.executorService);
0:         executorService.submit(
1:             new SortIteratorThread(iterators[i], sortDataRow, batchSize, rowCounter,
1:                 this.threadStatusObserver));
1:       checkError();
1:     checkError();
/////////////////////////////////////////////////////////////////////////
1:     private ThreadStatusObserver threadStatusObserver;
0: 
1:         UnsafeSortDataRows sortDataRows, int batchSize, AtomicLong rowCounter,
1:         ThreadStatusObserver threadStatusObserver) {
1:       this.threadStatusObserver = threadStatusObserver;
/////////////////////////////////////////////////////////////////////////
1:         this.threadStatusObserver.notifyFailed(e);
author:QiangCai
-------------------------------------------------------------------------------
commit:c5aba5f
/////////////////////////////////////////////////////////////////////////
1:         CarbonRowBatch rowBatch = new CarbonRowBatch(batchSize);
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
============================================================================
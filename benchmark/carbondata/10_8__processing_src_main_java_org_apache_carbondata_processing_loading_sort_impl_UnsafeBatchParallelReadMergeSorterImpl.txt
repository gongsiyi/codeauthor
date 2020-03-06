1:b13ead9: /*
1:b13ead9:  * Licensed to the Apache Software Foundation (ASF) under one or more
1:b13ead9:  * contributor license agreements.  See the NOTICE file distributed with
1:b13ead9:  * this work for additional information regarding copyright ownership.
1:b13ead9:  * The ASF licenses this file to You under the Apache License, Version 2.0
1:b13ead9:  * (the "License"); you may not use this file except in compliance with
1:b13ead9:  * the License.  You may obtain a copy of the License at
1:b13ead9:  *
1:b13ead9:  *    http://www.apache.org/licenses/LICENSE-2.0
1:b13ead9:  *
1:b13ead9:  * Unless required by applicable law or agreed to in writing, software
1:b13ead9:  * distributed under the License is distributed on an "AS IS" BASIS,
1:b13ead9:  * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:b13ead9:  * See the License for the specific language governing permissions and
1:b13ead9:  * limitations under the License.
1:b13ead9:  */
1:349c59c: package org.apache.carbondata.processing.loading.sort.impl;
2:b13ead9: 
1:00e0f7e: import java.io.File;
1:b13ead9: import java.util.Iterator;
1:b13ead9: import java.util.List;
1:b13ead9: import java.util.concurrent.BlockingQueue;
1:b13ead9: import java.util.concurrent.ExecutorService;
1:b13ead9: import java.util.concurrent.Executors;
1:b13ead9: import java.util.concurrent.LinkedBlockingQueue;
1:b13ead9: import java.util.concurrent.TimeUnit;
1:b13ead9: import java.util.concurrent.atomic.AtomicInteger;
1:b13ead9: import java.util.concurrent.atomic.AtomicLong;
1:b13ead9: 
1:b13ead9: import org.apache.carbondata.common.CarbonIterator;
1:b13ead9: import org.apache.carbondata.common.logging.LogService;
1:b13ead9: import org.apache.carbondata.common.logging.LogServiceFactory;
1:00e0f7e: import org.apache.carbondata.core.constants.CarbonCommonConstants;
1:dc83b2a: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1:dc83b2a: import org.apache.carbondata.core.datastore.row.CarbonRow;
1:b13ead9: import org.apache.carbondata.core.util.CarbonProperties;
1:b13ead9: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
1:349c59c: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1:349c59c: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1:349c59c: import org.apache.carbondata.processing.loading.row.CarbonSortBatch;
1:349c59c: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeCarbonRowPage;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeSortDataRows;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeIntermediateMerger;
1:349c59c: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
1:349c59c: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1:349c59c: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
1:00e0f7e: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
1:b13ead9: 
1:b13ead9: /**
1:b13ead9:  * It parallely reads data from array of iterates and do merge sort.
1:b13ead9:  * It sorts data in batches and send to the next step.
1:b13ead9:  */
1:53accb3: public class UnsafeBatchParallelReadMergeSorterImpl extends AbstractMergeSorter {
1:b13ead9: 
1:b13ead9:   private static final LogService LOGGER =
1:b13ead9:       LogServiceFactory.getLogService(UnsafeBatchParallelReadMergeSorterImpl.class.getName());
1:b13ead9: 
1:b13ead9:   private SortParameters sortParameters;
1:b13ead9: 
1:b13ead9:   private ExecutorService executorService;
1:b13ead9: 
1:b13ead9:   private AtomicLong rowCounter;
1:b13ead9: 
1:50248f5:   /* will be incremented for each batch. This ID is used in sort temp files name,
1:50248f5:    to identify files of that batch */
1:50248f5:   private AtomicInteger batchId;
1:50248f5: 
1:b13ead9:   public UnsafeBatchParallelReadMergeSorterImpl(AtomicLong rowCounter) {
1:b13ead9:     this.rowCounter = rowCounter;
1:408de86:   }
1:b13ead9: 
1:b13ead9:   @Override public void initialize(SortParameters sortParameters) {
2:b13ead9:     this.sortParameters = sortParameters;
1:50248f5:     batchId = new AtomicInteger(0);
1:b13ead9: 
1:b13ead9:   }
1:b13ead9: 
1:b13ead9:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:b13ead9:       throws CarbonDataLoadingException {
1:b13ead9:     this.executorService = Executors.newFixedThreadPool(iterators.length);
1:53accb3:     this.threadStatusObserver = new ThreadStatusObserver(this.executorService);
1:b13ead9:     int batchSize = CarbonProperties.getInstance().getBatchSize();
1:53accb3:     final SortBatchHolder sortBatchHolder = new SortBatchHolder(sortParameters, iterators.length,
1:53accb3:         this.threadStatusObserver);
1:b13ead9: 
1:b13ead9:     try {
1:b13ead9:       for (int i = 0; i < iterators.length; i++) {
1:06b0d08:         executorService.execute(
1:53accb3:             new SortIteratorThread(iterators[i], sortBatchHolder, batchSize, rowCounter,
1:53accb3:                 this.threadStatusObserver));
1:b13ead9:       }
1:b13ead9:     } catch (Exception e) {
1:53accb3:       checkError();
1:b13ead9:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:b13ead9:     }
1:53accb3:     checkError();
1:b13ead9:     // Creates the iterator to read from merge sorter.
1:b13ead9:     Iterator<CarbonSortBatch> batchIterator = new CarbonIterator<CarbonSortBatch>() {
1:b13ead9: 
1:b13ead9:       @Override public boolean hasNext() {
1:b13ead9:         return sortBatchHolder.hasNext();
1:b13ead9:       }
1:b13ead9: 
1:b13ead9:       @Override public CarbonSortBatch next() {
1:b13ead9:         return new CarbonSortBatch(sortBatchHolder.next());
1:b13ead9:       }
1:b13ead9:     };
1:b13ead9:     return new Iterator[] { batchIterator };
1:b13ead9:   }
1:b13ead9: 
1:b13ead9:   @Override public void close() {
1:b13ead9:     executorService.shutdown();
1:b13ead9:     try {
1:b13ead9:       executorService.awaitTermination(2, TimeUnit.DAYS);
2:b13ead9:     } catch (InterruptedException e) {
1:b13ead9:       LOGGER.error(e);
1:b13ead9:     }
1:b13ead9:   }
1:b13ead9: 
1:b13ead9:   /**
1:b13ead9:    * This thread iterates the iterator and adds the rows
1:b13ead9:    */
1:06b0d08:   private static class SortIteratorThread implements Runnable {
1:b13ead9: 
1:b13ead9:     private Iterator<CarbonRowBatch> iterator;
1:b13ead9: 
1:b13ead9:     private SortBatchHolder sortDataRows;
1:b13ead9: 
1:b13ead9:     private Object[][] buffer;
1:b13ead9: 
1:b13ead9:     private AtomicLong rowCounter;
1:b13ead9: 
1:53accb3:     private ThreadStatusObserver threadStatusObserver;
1:b13ead9: 
1:b13ead9:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortBatchHolder sortDataRows,
1:53accb3:         int batchSize, AtomicLong rowCounter, ThreadStatusObserver threadStatusObserver) {
1:b13ead9:       this.iterator = iterator;
1:b13ead9:       this.sortDataRows = sortDataRows;
1:b13ead9:       this.buffer = new Object[batchSize][];
1:b13ead9:       this.rowCounter = rowCounter;
1:53accb3:       this.threadStatusObserver = threadStatusObserver;
1:b13ead9:     }
1:b13ead9: 
1:06b0d08:     @Override
1:06b0d08:     public void run() {
1:b13ead9:       try {
1:b13ead9:         while (iterator.hasNext()) {
1:b13ead9:           CarbonRowBatch batch = iterator.next();
1:b13ead9:           int i = 0;
1:b13ead9:           while (batch.hasNext()) {
1:b13ead9:             CarbonRow row = batch.next();
1:b13ead9:             if (row != null) {
1:b13ead9:               buffer[i++] = row.getData();
1:b13ead9:             }
1:b13ead9:           }
1:b13ead9:           if (i > 0) {
1:b13ead9:             synchronized (sortDataRows) {
1:408de86:               sortDataRows.getSortDataRow().addRowBatchWithOutSync(buffer, i);
1:408de86:               rowCounter.getAndAdd(i);
1:b13ead9:               if (!sortDataRows.getSortDataRow().canAdd()) {
1:0205fa6:                 sortDataRows.finish(false);
1:b13ead9:                 sortDataRows.createSortDataRows();
1:b13ead9:               }
1:b13ead9:             }
1:b13ead9:           }
1:b13ead9:         }
1:b13ead9:       } catch (Exception e) {
1:b13ead9:         LOGGER.error(e);
1:53accb3:         this.threadStatusObserver.notifyFailed(e);
1:b13ead9:       } finally {
1:7ef9164:         synchronized (sortDataRows) {
1:7ef9164:           sortDataRows.finishThread();
1:7ef9164:         }
1:b13ead9:       }
1:b13ead9:     }
1:b13ead9: 
1:b13ead9:   }
1:b13ead9: 
1:50248f5:   private class SortBatchHolder
1:b13ead9:       extends CarbonIterator<UnsafeSingleThreadFinalSortFilesMerger> {
1:b13ead9: 
1:b13ead9:     private SortParameters sortParameters;
1:b13ead9: 
1:b13ead9:     private UnsafeSingleThreadFinalSortFilesMerger finalMerger;
1:b13ead9: 
1:b13ead9:     private UnsafeIntermediateMerger unsafeIntermediateFileMerger;
1:b13ead9: 
1:b13ead9:     private UnsafeSortDataRows sortDataRow;
1:b13ead9: 
1:b13ead9:     private final BlockingQueue<UnsafeSingleThreadFinalSortFilesMerger> mergerQueue;
1:b13ead9: 
1:b13ead9:     private AtomicInteger iteratorCount;
1:b13ead9: 
1:00e0f7e:     private int batchCount;
1:00e0f7e: 
1:53accb3:     private ThreadStatusObserver threadStatusObserver;
1:b13ead9: 
1:00e0f7e:     private final Object lock = new Object();
1:00e0f7e: 
1:50248f5:     SortBatchHolder(SortParameters sortParameters, int numberOfThreads,
1:53accb3:         ThreadStatusObserver threadStatusObserver) {
1:00e0f7e:       this.sortParameters = sortParameters.getCopy();
1:b13ead9:       this.iteratorCount = new AtomicInteger(numberOfThreads);
1:72235b3:       this.mergerQueue = new LinkedBlockingQueue<>(1);
1:53accb3:       this.threadStatusObserver = threadStatusObserver;
1:b13ead9:       createSortDataRows();
1:b13ead9:     }
1:b13ead9: 
1:b13ead9:     private void createSortDataRows() {
1:50248f5:       // For each batch, createSortDataRows() will be called.
1:50248f5:       // Files saved to disk during sorting of previous batch,should not be considered
1:50248f5:       // for this batch.
1:50248f5:       // Hence use batchID as rangeID field of sorttempfiles.
1:50248f5:       // so getFilesToMergeSort() will select only this batch files.
1:50248f5:       this.sortParameters.setRangeId(batchId.incrementAndGet());
1:f82b10b:       int inMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
1:00e0f7e:       setTempLocation(sortParameters);
1:f82b10b:       this.finalMerger = new UnsafeSingleThreadFinalSortFilesMerger(sortParameters,
1:f82b10b:           sortParameters.getTempFileLocation());
1:b13ead9:       unsafeIntermediateFileMerger = new UnsafeIntermediateMerger(sortParameters);
1:f82b10b:       sortDataRow = new UnsafeSortDataRows(sortParameters, unsafeIntermediateFileMerger,
1:f82b10b:           inMemoryChunkSizeInMB);
1:b13ead9: 
1:b13ead9:       try {
1:b13ead9:         sortDataRow.initialize();
1:b439b00:       } catch (Exception e) {
2:b13ead9:         throw new CarbonDataLoadingException(e);
1:b13ead9:       }
1:00e0f7e:       batchCount++;
1:00e0f7e:     }
1:00e0f7e: 
1:00e0f7e:     private void setTempLocation(SortParameters parameters) {
1:5bedd77:       String[] carbonDataDirectoryPath = CarbonDataProcessorUtil.getLocalDataFolderLocation(
1:5bedd77:           parameters.getDatabaseName(), parameters.getTableName(), parameters.getTaskNo(),
1:5bedd77:           parameters.getSegmentId(), false, false);
1:ded8b41:       String[] tempDirs = CarbonDataProcessorUtil.arrayAppend(carbonDataDirectoryPath,
1:ded8b41:           File.separator, CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:ded8b41:       parameters.setTempFileLocation(tempDirs);
1:b13ead9:     }
1:b13ead9: 
1:b13ead9:     @Override public UnsafeSingleThreadFinalSortFilesMerger next() {
1:b13ead9:       try {
1:53accb3:         UnsafeSingleThreadFinalSortFilesMerger unsafeSingleThreadFinalSortFilesMerger =
1:53accb3:             mergerQueue.take();
1:53accb3:         if (unsafeSingleThreadFinalSortFilesMerger.isStopProcess()) {
1:53accb3:           throw new RuntimeException(threadStatusObserver.getThrowable());
1:b13ead9:         }
1:53accb3:         return unsafeSingleThreadFinalSortFilesMerger;
1:b13ead9:       } catch (InterruptedException e) {
1:b13ead9:         throw new RuntimeException(e);
1:b13ead9:       }
1:b13ead9:     }
1:b13ead9: 
1:b13ead9:     public UnsafeSortDataRows getSortDataRow() {
1:b13ead9:       return sortDataRow;
1:b13ead9:     }
1:b13ead9: 
1:0205fa6:     public void finish(boolean isFinalAttempt) {
1:b13ead9:       try {
1:53accb3:         // if the mergerQue is empty and some CarbonDataLoadingException exception has occurred
1:53accb3:         // then set stop process to true in the finalmerger instance
1:53accb3:         if (mergerQueue.isEmpty() && threadStatusObserver != null
1:53accb3:             && threadStatusObserver.getThrowable() != null && threadStatusObserver
1:53accb3:             .getThrowable() instanceof CarbonDataLoadingException) {
1:53accb3:           finalMerger.setStopProcess(true);
1:0205fa6:           if (isFinalAttempt) {
1:0205fa6:             iteratorCount.decrementAndGet();
1:0205fa6:           }
1:00e0f7e:           mergerQueue.put(finalMerger);
1:72235b3:           return;
1:b13ead9:         }
1:b13ead9:         processRowToNextStep(sortDataRow, sortParameters);
1:b13ead9:         unsafeIntermediateFileMerger.finish();
1:b13ead9:         List<UnsafeCarbonRowPage> rowPages = unsafeIntermediateFileMerger.getRowPages();
1:b13ead9:         finalMerger.startFinalMerge(rowPages.toArray(new UnsafeCarbonRowPage[rowPages.size()]),
1:b13ead9:             unsafeIntermediateFileMerger.getMergedPages());
1:b13ead9:         unsafeIntermediateFileMerger.close();
1:0205fa6:         if (isFinalAttempt) {
1:0205fa6:           iteratorCount.decrementAndGet();
1:0205fa6:         }
1:00e0f7e:         mergerQueue.put(finalMerger);
1:b13ead9:         sortDataRow = null;
1:b13ead9:         unsafeIntermediateFileMerger = null;
1:b13ead9:         finalMerger = null;
1:b13ead9:       } catch (CarbonDataWriterException e) {
1:b13ead9:         throw new CarbonDataLoadingException(e);
2:b13ead9:       } catch (CarbonSortKeyAndGroupByException e) {
1:00e0f7e:         throw new CarbonDataLoadingException(e);
1:00e0f7e:       } catch (InterruptedException e) {
1:72235b3:         // if fails to put in queue because of interrupted exception, we can offer to free the main
1:72235b3:         // thread from waiting.
1:72235b3:         if (finalMerger != null) {
1:72235b3:           finalMerger.setStopProcess(true);
1:06b0d08:           boolean offered = mergerQueue.offer(finalMerger);
1:06b0d08:           if (!offered) {
1:06b0d08:             throw new CarbonDataLoadingException(e);
1:06b0d08:           }
1:72235b3:         }
1:b13ead9:         throw new CarbonDataLoadingException(e);
1:b13ead9:       }
1:b13ead9:     }
1:b13ead9: 
1:00e0f7e:     public void finishThread() {
1:00e0f7e:       synchronized (lock) {
1:0205fa6:         if (iteratorCount.get() <= 1) {
1:0205fa6:           finish(true);
1:0205fa6:         } else {
1:0205fa6:           iteratorCount.decrementAndGet();
1:00e0f7e:         }
1:b13ead9:       }
1:b13ead9:     }
1:b13ead9: 
1:00e0f7e:     public boolean hasNext() {
1:b13ead9:       return iteratorCount.get() > 0 || !mergerQueue.isEmpty();
1:b13ead9:     }
1:b13ead9: 
1:b13ead9:     /**
1:b13ead9:      * Below method will be used to process data to next step
1:b13ead9:      */
1:b13ead9:     private boolean processRowToNextStep(UnsafeSortDataRows sortDataRows, SortParameters parameters)
1:b13ead9:         throws CarbonDataLoadingException {
1:b13ead9:       try {
1:b13ead9:         // start sorting
1:b13ead9:         sortDataRows.startSorting();
1:b13ead9: 
1:b13ead9:         // check any more rows are present
2:b13ead9:         LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:b13ead9:         CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:b13ead9:             .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:b13ead9:         CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:b13ead9:             .recordDictionaryValuesTotalTime(parameters.getPartitionID(),
1:b13ead9:                 System.currentTimeMillis());
2:b13ead9:         return false;
1:b439b00:       } catch (Exception e) {
1:b13ead9:         throw new CarbonDataLoadingException(e);
1:b13ead9:       }
1:b13ead9:     }
1:b13ead9:   }
1:b13ead9: }
============================================================================
author:ajantha-bhat
-------------------------------------------------------------------------------
commit:50248f5
/////////////////////////////////////////////////////////////////////////
1:   /* will be incremented for each batch. This ID is used in sort temp files name,
1:    to identify files of that batch */
1:   private AtomicInteger batchId;
1: 
1:     batchId = new AtomicInteger(0);
/////////////////////////////////////////////////////////////////////////
1:   private class SortBatchHolder
/////////////////////////////////////////////////////////////////////////
1:     SortBatchHolder(SortParameters sortParameters, int numberOfThreads,
/////////////////////////////////////////////////////////////////////////
1:       // For each batch, createSortDataRows() will be called.
1:       // Files saved to disk during sorting of previous batch,should not be considered
1:       // for this batch.
1:       // Hence use batchID as rangeID field of sorttempfiles.
1:       // so getFilesToMergeSort() will select only this batch files.
1:       this.sortParameters.setRangeId(batchId.incrementAndGet());
author:sraghunandan
-------------------------------------------------------------------------------
commit:f911403
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
author:Raghunandan S
-------------------------------------------------------------------------------
commit:7ef9164
/////////////////////////////////////////////////////////////////////////
1:         synchronized (sortDataRows) {
1:           sortDataRows.finishThread();
1:         }
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
/////////////////////////////////////////////////////////////////////////
1:           boolean offered = mergerQueue.offer(finalMerger);
1:           if (!offered) {
1:             throw new CarbonDataLoadingException(e);
1:           }
author:xuchuanyin
-------------------------------------------------------------------------------
commit:b439b00
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:       } catch (Exception e) {
/////////////////////////////////////////////////////////////////////////
1:       } catch (Exception e) {
commit:ded8b41
/////////////////////////////////////////////////////////////////////////
0:       String[] carbonDataDirectoryPath = CarbonDataProcessorUtil
1:       String[] tempDirs = CarbonDataProcessorUtil.arrayAppend(carbonDataDirectoryPath,
1:           File.separator, CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
1:       parameters.setTempFileLocation(tempDirs);
author:Jacky Li
-------------------------------------------------------------------------------
commit:5bedd77
/////////////////////////////////////////////////////////////////////////
1:       String[] carbonDataDirectoryPath = CarbonDataProcessorUtil.getLocalDataFolderLocation(
1:           parameters.getDatabaseName(), parameters.getTableName(), parameters.getTaskNo(),
1:           parameters.getSegmentId(), false, false);
commit:349c59c
/////////////////////////////////////////////////////////////////////////
1: package org.apache.carbondata.processing.loading.sort.impl;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.processing.loading.exception.CarbonDataLoadingException;
1: import org.apache.carbondata.processing.loading.row.CarbonRowBatch;
1: import org.apache.carbondata.processing.loading.row.CarbonSortBatch;
1: import org.apache.carbondata.processing.loading.sort.AbstractMergeSorter;
1: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeCarbonRowPage;
1: import org.apache.carbondata.processing.loading.sort.unsafe.UnsafeSortDataRows;
1: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeIntermediateMerger;
1: import org.apache.carbondata.processing.loading.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
1: import org.apache.carbondata.processing.sort.exception.CarbonSortKeyAndGroupByException;
1: import org.apache.carbondata.processing.sort.sortdata.SortParameters;
author:lionelcao
-------------------------------------------------------------------------------
commit:874764f
/////////////////////////////////////////////////////////////////////////
0:             parameters.getSegmentId(), false, false);
author:dhatchayani
-------------------------------------------------------------------------------
commit:0205fa6
/////////////////////////////////////////////////////////////////////////
1:                 sortDataRows.finish(false);
/////////////////////////////////////////////////////////////////////////
1:     public void finish(boolean isFinalAttempt) {
/////////////////////////////////////////////////////////////////////////
1:           if (isFinalAttempt) {
1:             iteratorCount.decrementAndGet();
1:           }
/////////////////////////////////////////////////////////////////////////
1:         if (isFinalAttempt) {
1:           iteratorCount.decrementAndGet();
1:         }
/////////////////////////////////////////////////////////////////////////
1:         if (iteratorCount.get() <= 1) {
1:           finish(true);
1:         } else {
1:           iteratorCount.decrementAndGet();
commit:00e0f7e
/////////////////////////////////////////////////////////////////////////
1: import java.io.File;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.constants.CarbonCommonConstants;
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.processing.util.CarbonDataProcessorUtil;
/////////////////////////////////////////////////////////////////////////
1:     private int batchCount;
1: 
1:     private final Object lock = new Object();
1: 
1:       this.sortParameters = sortParameters.getCopy();
/////////////////////////////////////////////////////////////////////////
1:       setTempLocation(sortParameters);
/////////////////////////////////////////////////////////////////////////
1:       batchCount++;
1:     }
1: 
1:     private void setTempLocation(SortParameters parameters) {
0:       String carbonDataDirectoryPath = CarbonDataProcessorUtil
0:           .getLocalDataFolderLocation(parameters.getDatabaseName(),
0:             parameters.getTableName(), parameters.getTaskNo(), batchCount + "",
0:             parameters.getSegmentId(), false);
0:       parameters.setTempFileLocation(
0:           carbonDataDirectoryPath + File.separator + CarbonCommonConstants.SORT_TEMP_FILE_LOCATION);
/////////////////////////////////////////////////////////////////////////
1:           mergerQueue.put(finalMerger);
/////////////////////////////////////////////////////////////////////////
1:         mergerQueue.put(finalMerger);
/////////////////////////////////////////////////////////////////////////
1:       } catch (InterruptedException e) {
1:         throw new CarbonDataLoadingException(e);
1:     public void finishThread() {
1:       synchronized (lock) {
0:         if (iteratorCount.decrementAndGet() <= 0) {
0:           finish();
1:         }
1:     public boolean hasNext() {
commit:408de86
/////////////////////////////////////////////////////////////////////////
1:               sortDataRows.getSortDataRow().addRowBatchWithOutSync(buffer, i);
1:               rowCounter.getAndAdd(i);
/////////////////////////////////////////////////////////////////////////
0:       if (inMemoryChunkSizeInMB > sortParameters.getBatchSortSizeinMb()) {
0:         inMemoryChunkSizeInMB = sortParameters.getBatchSortSizeinMb();
1:       }
author:ravipesala
-------------------------------------------------------------------------------
commit:72235b3
/////////////////////////////////////////////////////////////////////////
1:       this.mergerQueue = new LinkedBlockingQueue<>(1);
/////////////////////////////////////////////////////////////////////////
1:           return;
/////////////////////////////////////////////////////////////////////////
1:         // if fails to put in queue because of interrupted exception, we can offer to free the main
1:         // thread from waiting.
1:         if (finalMerger != null) {
1:           finalMerger.setStopProcess(true);
0:           mergerQueue.offer(finalMerger);
1:         }
commit:9d29083
/////////////////////////////////////////////////////////////////////////
commit:f82b10b
/////////////////////////////////////////////////////////////////////////
1:       int inMemoryChunkSizeInMB = CarbonProperties.getInstance().getSortMemoryChunkSizeInMB();
1:       this.finalMerger = new UnsafeSingleThreadFinalSortFilesMerger(sortParameters,
1:           sortParameters.getTempFileLocation());
1:       sortDataRow = new UnsafeSortDataRows(sortParameters, unsafeIntermediateFileMerger,
1:           inMemoryChunkSizeInMB);
commit:b13ead9
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
0: package org.apache.carbondata.processing.newflow.sort.impl;
1: 
1: import java.util.Iterator;
1: import java.util.List;
1: import java.util.concurrent.BlockingQueue;
0: import java.util.concurrent.Callable;
1: import java.util.concurrent.ExecutorService;
1: import java.util.concurrent.Executors;
1: import java.util.concurrent.LinkedBlockingQueue;
1: import java.util.concurrent.TimeUnit;
1: import java.util.concurrent.atomic.AtomicInteger;
1: import java.util.concurrent.atomic.AtomicLong;
1: 
1: import org.apache.carbondata.common.CarbonIterator;
1: import org.apache.carbondata.common.logging.LogService;
1: import org.apache.carbondata.common.logging.LogServiceFactory;
1: import org.apache.carbondata.core.util.CarbonProperties;
1: import org.apache.carbondata.core.util.CarbonTimeStatisticsFactory;
0: import org.apache.carbondata.processing.newflow.exception.CarbonDataLoadingException;
0: import org.apache.carbondata.processing.newflow.row.CarbonRow;
0: import org.apache.carbondata.processing.newflow.row.CarbonRowBatch;
0: import org.apache.carbondata.processing.newflow.row.CarbonSortBatch;
0: import org.apache.carbondata.processing.newflow.sort.Sorter;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.UnsafeCarbonRowPage;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.UnsafeSortDataRows;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.merger.UnsafeIntermediateMerger;
0: import org.apache.carbondata.processing.newflow.sort.unsafe.merger.UnsafeSingleThreadFinalSortFilesMerger;
0: import org.apache.carbondata.processing.sortandgroupby.exception.CarbonSortKeyAndGroupByException;
0: import org.apache.carbondata.processing.sortandgroupby.sortdata.SortParameters;
0: import org.apache.carbondata.processing.store.writer.exception.CarbonDataWriterException;
1: 
1: /**
1:  * It parallely reads data from array of iterates and do merge sort.
1:  * It sorts data in batches and send to the next step.
1:  */
0: public class UnsafeBatchParallelReadMergeSorterImpl implements Sorter {
1: 
1:   private static final LogService LOGGER =
1:       LogServiceFactory.getLogService(UnsafeBatchParallelReadMergeSorterImpl.class.getName());
1: 
1:   private SortParameters sortParameters;
1: 
1:   private ExecutorService executorService;
1: 
1:   private AtomicLong rowCounter;
1: 
1:   public UnsafeBatchParallelReadMergeSorterImpl(AtomicLong rowCounter) {
1:     this.rowCounter = rowCounter;
1:   }
1: 
1:   @Override public void initialize(SortParameters sortParameters) {
1:     this.sortParameters = sortParameters;
1: 
1:   }
1: 
1:   @Override public Iterator<CarbonRowBatch>[] sort(Iterator<CarbonRowBatch>[] iterators)
1:       throws CarbonDataLoadingException {
1:     this.executorService = Executors.newFixedThreadPool(iterators.length);
1:     int batchSize = CarbonProperties.getInstance().getBatchSize();
0:     final SortBatchHolder sortBatchHolder = new SortBatchHolder(sortParameters, iterators.length);
1: 
1:     try {
1:       for (int i = 0; i < iterators.length; i++) {
0:         executorService
0:             .submit(new SortIteratorThread(iterators[i], sortBatchHolder, batchSize, rowCounter));
1:       }
1:     } catch (Exception e) {
1:       throw new CarbonDataLoadingException("Problem while shutdown the server ", e);
1:     }
1: 
1:     // Creates the iterator to read from merge sorter.
1:     Iterator<CarbonSortBatch> batchIterator = new CarbonIterator<CarbonSortBatch>() {
1: 
1:       @Override public boolean hasNext() {
1:         return sortBatchHolder.hasNext();
1:       }
1: 
1:       @Override public CarbonSortBatch next() {
1:         return new CarbonSortBatch(sortBatchHolder.next());
1:       }
1:     };
1:     return new Iterator[] { batchIterator };
1:   }
1: 
1:   @Override public void close() {
1:     executorService.shutdown();
1:     try {
1:       executorService.awaitTermination(2, TimeUnit.DAYS);
1:     } catch (InterruptedException e) {
1:       LOGGER.error(e);
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
1:     private SortBatchHolder sortDataRows;
1: 
1:     private Object[][] buffer;
1: 
1:     private AtomicLong rowCounter;
1: 
1:     public SortIteratorThread(Iterator<CarbonRowBatch> iterator, SortBatchHolder sortDataRows,
0:         int batchSize, AtomicLong rowCounter) {
1:       this.iterator = iterator;
1:       this.sortDataRows = sortDataRows;
1:       this.buffer = new Object[batchSize][];
1:       this.rowCounter = rowCounter;
1:     }
1: 
0:     @Override public Void call() throws CarbonDataLoadingException {
1:       try {
1:         while (iterator.hasNext()) {
1:           CarbonRowBatch batch = iterator.next();
1:           int i = 0;
1:           while (batch.hasNext()) {
1:             CarbonRow row = batch.next();
1:             if (row != null) {
1:               buffer[i++] = row.getData();
1:             }
1:           }
1:           if (i > 0) {
0:             sortDataRows.getSortDataRow().addRowBatch(buffer, i);
0:             rowCounter.getAndAdd(i);
1:             synchronized (sortDataRows) {
1:               if (!sortDataRows.getSortDataRow().canAdd()) {
0:                 sortDataRows.finish();
1:                 sortDataRows.createSortDataRows();
1:               }
1:             }
1:           }
1:         }
1:       } catch (Exception e) {
1:         LOGGER.error(e);
1:         throw new CarbonDataLoadingException(e);
1:       } finally {
0:         sortDataRows.finishThread();
1:       }
0:       return null;
1:     }
1: 
1:   }
1: 
0:   private static class SortBatchHolder
1:       extends CarbonIterator<UnsafeSingleThreadFinalSortFilesMerger> {
1: 
1:     private SortParameters sortParameters;
1: 
1:     private UnsafeSingleThreadFinalSortFilesMerger finalMerger;
1: 
1:     private UnsafeIntermediateMerger unsafeIntermediateFileMerger;
1: 
1:     private UnsafeSortDataRows sortDataRow;
1: 
1:     private final BlockingQueue<UnsafeSingleThreadFinalSortFilesMerger> mergerQueue;
1: 
1:     private AtomicInteger iteratorCount;
1: 
0:     public SortBatchHolder(SortParameters sortParameters, int numberOfThreads) {
1:       this.sortParameters = sortParameters;
1:       this.iteratorCount = new AtomicInteger(numberOfThreads);
0:       this.mergerQueue = new LinkedBlockingQueue<>();
1:       createSortDataRows();
1:     }
1: 
1:     private void createSortDataRows() {
0:       this.finalMerger = new UnsafeSingleThreadFinalSortFilesMerger(sortParameters);
1:       unsafeIntermediateFileMerger = new UnsafeIntermediateMerger(sortParameters);
0:       sortDataRow = new UnsafeSortDataRows(sortParameters, unsafeIntermediateFileMerger);
1: 
1:       try {
1:         sortDataRow.initialize();
1:       } catch (CarbonSortKeyAndGroupByException e) {
1:         throw new CarbonDataLoadingException(e);
1:       }
1:     }
1: 
1:     @Override public UnsafeSingleThreadFinalSortFilesMerger next() {
1:       try {
0:         return mergerQueue.take();
1:       } catch (InterruptedException e) {
1:         throw new RuntimeException(e);
1:       }
1:     }
1: 
1:     public UnsafeSortDataRows getSortDataRow() {
1:       return sortDataRow;
1:     }
1: 
0:     public void finish() {
1:       try {
1:         processRowToNextStep(sortDataRow, sortParameters);
1:         unsafeIntermediateFileMerger.finish();
1:         List<UnsafeCarbonRowPage> rowPages = unsafeIntermediateFileMerger.getRowPages();
1:         finalMerger.startFinalMerge(rowPages.toArray(new UnsafeCarbonRowPage[rowPages.size()]),
1:             unsafeIntermediateFileMerger.getMergedPages());
1:         unsafeIntermediateFileMerger.close();
0:         mergerQueue.offer(finalMerger);
1:         sortDataRow = null;
1:         unsafeIntermediateFileMerger = null;
1:         finalMerger = null;
1:       } catch (CarbonDataWriterException e) {
1:         throw new CarbonDataLoadingException(e);
1:       } catch (CarbonSortKeyAndGroupByException e) {
1:         throw new CarbonDataLoadingException(e);
1:       }
1:     }
1: 
0:     public synchronized void finishThread() {
0:       if (iteratorCount.decrementAndGet() <= 0) {
0:         finish();
1:       }
1:     }
1: 
0:     public synchronized boolean hasNext() {
1:       return iteratorCount.get() > 0 || !mergerQueue.isEmpty();
1:     }
1: 
1:     /**
1:      * Below method will be used to process data to next step
1:      */
1:     private boolean processRowToNextStep(UnsafeSortDataRows sortDataRows, SortParameters parameters)
1:         throws CarbonDataLoadingException {
0:       if (null == sortDataRows) {
1:         LOGGER.info("Record Processed For table: " + parameters.getTableName());
0:         LOGGER.info("Number of Records was Zero");
0:         String logMessage = "Summary: Carbon Sort Key Step: Read: " + 0 + ": Write: " + 0;
0:         LOGGER.info(logMessage);
1:         return false;
1:       }
1: 
1:       try {
1:         // start sorting
1:         sortDataRows.startSorting();
1: 
1:         // check any more rows are present
1:         LOGGER.info("Record Processed For table: " + parameters.getTableName());
1:         CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:             .recordSortRowsStepTotalTime(parameters.getPartitionID(), System.currentTimeMillis());
1:         CarbonTimeStatisticsFactory.getLoadStatisticsInstance()
1:             .recordDictionaryValuesTotalTime(parameters.getPartitionID(),
1:                 System.currentTimeMillis());
1:         return false;
1:       } catch (InterruptedException e) {
1:         throw new CarbonDataLoadingException(e);
1:       }
1:     }
1: 
1:   }
1: }
author:jackylk
-------------------------------------------------------------------------------
commit:edda248
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.core.memory.MemoryException;
/////////////////////////////////////////////////////////////////////////
0:       } catch (MemoryException e) {
commit:dc83b2a
/////////////////////////////////////////////////////////////////////////
1: import org.apache.carbondata.core.datastore.exception.CarbonDataWriterException;
1: import org.apache.carbondata.core.datastore.row.CarbonRow;
/////////////////////////////////////////////////////////////////////////
author:mohammadshahidkhan
-------------------------------------------------------------------------------
commit:53accb3
/////////////////////////////////////////////////////////////////////////
0: import org.apache.carbondata.processing.newflow.sort.AbstractMergeSorter;
/////////////////////////////////////////////////////////////////////////
1: public class UnsafeBatchParallelReadMergeSorterImpl extends AbstractMergeSorter {
/////////////////////////////////////////////////////////////////////////
1:     this.threadStatusObserver = new ThreadStatusObserver(this.executorService);
1:     final SortBatchHolder sortBatchHolder = new SortBatchHolder(sortParameters, iterators.length,
1:         this.threadStatusObserver);
0:         executorService.submit(
1:             new SortIteratorThread(iterators[i], sortBatchHolder, batchSize, rowCounter,
1:                 this.threadStatusObserver));
1:       checkError();
1:     checkError();
/////////////////////////////////////////////////////////////////////////
1:     private ThreadStatusObserver threadStatusObserver;
0: 
1:         int batchSize, AtomicLong rowCounter, ThreadStatusObserver threadStatusObserver) {
1:       this.threadStatusObserver = threadStatusObserver;
/////////////////////////////////////////////////////////////////////////
1:         this.threadStatusObserver.notifyFailed(e);
/////////////////////////////////////////////////////////////////////////
1:     private ThreadStatusObserver threadStatusObserver;
0: 
0:     public SortBatchHolder(SortParameters sortParameters, int numberOfThreads,
1:         ThreadStatusObserver threadStatusObserver) {
1:       this.threadStatusObserver = threadStatusObserver;
/////////////////////////////////////////////////////////////////////////
1:         UnsafeSingleThreadFinalSortFilesMerger unsafeSingleThreadFinalSortFilesMerger =
1:             mergerQueue.take();
1:         if (unsafeSingleThreadFinalSortFilesMerger.isStopProcess()) {
1:           throw new RuntimeException(threadStatusObserver.getThrowable());
0:         }
1:         return unsafeSingleThreadFinalSortFilesMerger;
/////////////////////////////////////////////////////////////////////////
1:         // if the mergerQue is empty and some CarbonDataLoadingException exception has occurred
1:         // then set stop process to true in the finalmerger instance
1:         if (mergerQueue.isEmpty() && threadStatusObserver != null
1:             && threadStatusObserver.getThrowable() != null && threadStatusObserver
1:             .getThrowable() instanceof CarbonDataLoadingException) {
1:           finalMerger.setStopProcess(true);
0:           mergerQueue.offer(finalMerger);
0:         }
============================================================================
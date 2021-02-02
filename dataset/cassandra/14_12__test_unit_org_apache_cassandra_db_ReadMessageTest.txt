1:88edbed: /*
1:88edbed: * Licensed to the Apache Software Foundation (ASF) under one
1:88edbed: * or more contributor license agreements.  See the NOTICE file
1:88edbed: * distributed with this work for additional information
1:88edbed: * regarding copyright ownership.  The ASF licenses this file
1:88edbed: * to you under the Apache License, Version 2.0 (the
1:88edbed: * "License"); you may not use this file except in compliance
1:88edbed: * with the License.  You may obtain a copy of the License at
1:88edbed: *
1:88edbed: *    http://www.apache.org/licenses/LICENSE-2.0
1:88edbed: *
1:88edbed: * Unless required by applicable law or agreed to in writing,
1:88edbed: * software distributed under the License is distributed on an
1:88edbed: * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
1:88edbed: * KIND, either express or implied.  See the License for the
1:88edbed: * specific language governing permissions and limitations
1:88edbed: * under the License.
1:88edbed: */
1:6931092: package org.apache.cassandra.db;
1:07cdfd0: 
1:a991b64: import static org.junit.Assert.*;
1:2fd3268: 
1:ea6ec42: import java.io.*;
1:2fd3268: 
1:36958f3: import com.google.common.base.Predicate;
1:03f72ac: 
1:d2a3827: import org.junit.BeforeClass;
1:e7a385a: import org.junit.Test;
1:7a75b63: import org.apache.cassandra.SchemaLoader;
1:e7a385a: import org.apache.cassandra.Util;
1:a991b64: import org.apache.cassandra.config.CFMetaData;
1:a991b64: import org.apache.cassandra.config.ColumnDefinition;
1:9797511: import org.apache.cassandra.config.DatabaseDescriptor;
1:36958f3: import org.apache.cassandra.db.commitlog.CommitLogTestReplayer;
1:a991b64: import org.apache.cassandra.db.partitions.PartitionUpdate;
1:a991b64: import org.apache.cassandra.db.rows.Row;
1:a991b64: import org.apache.cassandra.db.marshal.AsciiType;
1:a991b64: import org.apache.cassandra.db.marshal.BytesType;
1:a991b64: import org.apache.cassandra.db.partitions.FilteredPartition;
1:d2a3827: import org.apache.cassandra.exceptions.ConfigurationException;
1:a991b64: import org.apache.cassandra.io.IVersionedSerializer;
1:c4c9eae: import org.apache.cassandra.io.util.DataInputBuffer;
1:03f72ac: import org.apache.cassandra.io.util.DataInputPlus;
1:3655e91: import org.apache.cassandra.io.util.DataOutputBuffer;
1:1ecdd7f: import org.apache.cassandra.net.MessagingService;
1:31e3f61: import org.apache.cassandra.schema.KeyspaceParams;
1:8358669: import org.apache.cassandra.utils.ByteBufferUtil;
1:2fd3268: 
1:d2a3827: public class ReadMessageTest
2:de6dba7: {
1:d2a3827:     private static final String KEYSPACE1 = "ReadMessageTest1";
1:d2a3827:     private static final String KEYSPACENOCOMMIT = "ReadMessageTest_NoCommit";
1:d2a3827:     private static final String CF = "Standard1";
1:a991b64:     private static final String CF_FOR_READ_TEST = "Standard2";
1:a991b64:     private static final String CF_FOR_COMMIT_TEST = "Standard3";
1:2fd3268: 
1:d2a3827:     @BeforeClass
1:d2a3827:     public static void defineSchema() throws ConfigurationException
1:d2a3827:     {
1:9797511:         DatabaseDescriptor.daemonInitialization();
1:e31e216: 
1:a991b64:         CFMetaData cfForReadMetadata = CFMetaData.Builder.create(KEYSPACE1, CF_FOR_READ_TEST)
1:a991b64:                                                             .addPartitionKey("key", BytesType.instance)
1:a991b64:                                                             .addClusteringColumn("col1", AsciiType.instance)
1:a991b64:                                                             .addClusteringColumn("col2", AsciiType.instance)
1:a991b64:                                                             .addRegularColumn("a", AsciiType.instance)
1:a991b64:                                                             .addRegularColumn("b", AsciiType.instance).build();
1:a991b64: 
1:a991b64:         CFMetaData cfForCommitMetadata1 = CFMetaData.Builder.create(KEYSPACE1, CF_FOR_COMMIT_TEST)
1:a991b64:                                                        .addPartitionKey("key", BytesType.instance)
1:a991b64:                                                        .addClusteringColumn("name", AsciiType.instance)
1:a991b64:                                                        .addRegularColumn("commit1", AsciiType.instance).build();
1:a991b64: 
1:a991b64:         CFMetaData cfForCommitMetadata2 = CFMetaData.Builder.create(KEYSPACENOCOMMIT, CF_FOR_COMMIT_TEST)
1:a991b64:                                                             .addPartitionKey("key", BytesType.instance)
1:a991b64:                                                             .addClusteringColumn("name", AsciiType.instance)
1:a991b64:                                                             .addRegularColumn("commit2", AsciiType.instance).build();
1:a991b64: 
1:d2a3827:         SchemaLoader.prepareServer();
1:d2a3827:         SchemaLoader.createKeyspace(KEYSPACE1,
1:31e3f61:                                     KeyspaceParams.simple(1),
1:a991b64:                                     SchemaLoader.standardCFMD(KEYSPACE1, CF),
1:a991b64:                                     cfForReadMetadata,
1:a991b64:                                     cfForCommitMetadata1);
1:d2a3827:         SchemaLoader.createKeyspace(KEYSPACENOCOMMIT,
1:31e3f61:                                     KeyspaceParams.simpleTransient(1),
1:a991b64:                                     SchemaLoader.standardCFMD(KEYSPACENOCOMMIT, CF),
1:a991b64:                                     cfForCommitMetadata2);
1:d2a3827:     }
1:07cdfd0: 
1:07cdfd0:     @Test
1:986cee6:     public void testMakeReadMessage() throws IOException
1:de6dba7:     {
1:a991b64:         ColumnFamilyStore cfs = Keyspace.open(KEYSPACE1).getColumnFamilyStore(CF_FOR_READ_TEST);
1:e2a4ea7:         ReadCommand rm, rm2;
1:362cc05: 
1:a991b64:         rm = Util.cmd(cfs, Util.dk("key1"))
1:a991b64:                  .includeRow("col1", "col2")
1:a991b64:                  .build();
2:7d2115b:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
1:07cdfd0: 
1:a991b64:         rm = Util.cmd(cfs, Util.dk("key1"))
1:a991b64:                  .includeRow("col1", "col2")
1:a991b64:                  .reverse()
1:a991b64:                  .build();
1:7d2115b:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
1:07cdfd0: 
1:a991b64:         rm = Util.cmd(cfs)
1:a991b64:                  .build();
1:a991b64:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
1:a991b64: 
1:a991b64:         rm = Util.cmd(cfs)
1:a991b64:                  .fromKeyIncl(ByteBufferUtil.bytes("key1"))
1:a991b64:                  .toKeyIncl(ByteBufferUtil.bytes("key2"))
1:a991b64:                  .build();
1:a991b64:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
1:a991b64: 
1:a991b64:         rm = Util.cmd(cfs)
1:a991b64:                  .columns("a")
1:a991b64:                  .build();
1:a991b64:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
1:a991b64: 
1:a991b64:         rm = Util.cmd(cfs)
1:a991b64:                  .includeRow("col1", "col2")
1:a991b64:                  .columns("a")
1:a991b64:                  .build();
1:a991b64:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
1:a991b64: 
1:a991b64:         rm = Util.cmd(cfs)
1:a991b64:                  .fromKeyIncl(ByteBufferUtil.bytes("key1"))
1:a991b64:                  .includeRow("col1", "col2")
1:a991b64:                  .columns("a")
1:a991b64:                  .build();
1:7d2115b:         rm2 = serializeAndDeserializeReadMessage(rm);
1:a991b64:         assertEquals(rm.toString(), rm2.toString());
2:de6dba7:     }
1:07cdfd0: 
1:986cee6:     private ReadCommand serializeAndDeserializeReadMessage(ReadCommand rm) throws IOException
1:de6dba7:     {
1:a991b64:         IVersionedSerializer<ReadCommand> rms = ReadCommand.serializer;
1:60d9c7f:         DataOutputBuffer out = new DataOutputBuffer();
1:07cdfd0: 
1:60d9c7f:         rms.serialize(rm, out, MessagingService.current_version);
1:a991b64: 
1:c4c9eae:         DataInputPlus dis = new DataInputBuffer(out.getData());
1:03f72ac:         return rms.deserialize(dis, MessagingService.current_version);
1:de6dba7:     }
1:a991b64: 
1:07cdfd0: 
1:2fd3268:     @Test
1:9639f95:     public void testGetColumn()
1:de6dba7:     {
1:a991b64:         ColumnFamilyStore cfs = Keyspace.open(KEYSPACE1).getColumnFamilyStore(CF);
1:07cdfd0: 
1:a991b64:         new RowUpdateBuilder(cfs.metadata, 0, ByteBufferUtil.bytes("key1"))
1:a991b64:                 .clustering("Column1")
1:a991b64:                 .add("val", ByteBufferUtil.bytes("abcd"))
1:a991b64:                 .build()
1:a991b64:                 .apply();
1:07cdfd0: 
1:a991b64:         ColumnDefinition col = cfs.metadata.getColumnDefinition(ByteBufferUtil.bytes("val"));
1:a991b64:         int found = 0;
1:a991b64:         for (FilteredPartition partition : Util.getAll(Util.cmd(cfs).build()))
1:a991b64:         {
1:a991b64:             for (Row r : partition)
1:a991b64:             {
1:a991b64:                 if (r.getCell(col).value().equals(ByteBufferUtil.bytes("abcd")))
1:a991b64:                     ++found;
1:a991b64:             }
1:a991b64:         }
1:a991b64:         assertEquals(1, found);
1:de6dba7:     }
1:07cdfd0: 
1:d14f814:     @Test
1:ea6ec42:     public void testNoCommitLog() throws Exception
1:de6dba7:     {
1:a991b64:         ColumnFamilyStore cfs = Keyspace.open(KEYSPACE1).getColumnFamilyStore(CF_FOR_COMMIT_TEST);
1:07cdfd0: 
1:a991b64:         ColumnFamilyStore cfsnocommit = Keyspace.open(KEYSPACENOCOMMIT).getColumnFamilyStore(CF_FOR_COMMIT_TEST);
1:07cdfd0: 
1:a991b64:         new RowUpdateBuilder(cfs.metadata, 0, ByteBufferUtil.bytes("row"))
1:a991b64:                 .clustering("c")
1:a991b64:                 .add("commit1", ByteBufferUtil.bytes("abcd"))
1:a991b64:                 .build()
1:a991b64:                 .apply();
1:a991b64: 
1:a991b64:         new RowUpdateBuilder(cfsnocommit.metadata, 0, ByteBufferUtil.bytes("row"))
1:a991b64:                 .clustering("c")
1:a991b64:                 .add("commit2", ByteBufferUtil.bytes("abcd"))
1:a991b64:                 .build()
1:a991b64:                 .apply();
1:a991b64: 
1:a991b64:         Checker checker = new Checker(cfs.metadata.getColumnDefinition(ByteBufferUtil.bytes("commit1")),
1:a991b64:                                       cfsnocommit.metadata.getColumnDefinition(ByteBufferUtil.bytes("commit2")));
1:e31e216: 
1:e31e216:         CommitLogTestReplayer replayer = new CommitLogTestReplayer(checker);
1:e31e216:         replayer.examineCommitLog();
1:07cdfd0: 
1:36958f3:         assertTrue(checker.commitLogMessageFound);
1:36958f3:         assertFalse(checker.noCommitLogMessageFound);
1:36958f3:     }
1:07cdfd0: 
1:36958f3:     static class Checker implements Predicate<Mutation>
1:36958f3:     {
1:a991b64:         private final ColumnDefinition withCommit;
1:a991b64:         private final ColumnDefinition withoutCommit;
1:a991b64: 
1:ea6ec42:         boolean commitLogMessageFound = false;
1:ea6ec42:         boolean noCommitLogMessageFound = false;
1:07cdfd0: 
1:a991b64:         public Checker(ColumnDefinition withCommit, ColumnDefinition withoutCommit)
1:a991b64:         {
1:a991b64:             this.withCommit = withCommit;
1:a991b64:             this.withoutCommit = withoutCommit;
1:a991b64:         }
1:a991b64: 
1:36958f3:         public boolean apply(Mutation mutation)
1:de6dba7:         {
1:a991b64:             for (PartitionUpdate upd : mutation.getPartitionUpdates())
1:de6dba7:             {
1:2f41243:                 Row r = upd.getRow(Clustering.make(ByteBufferUtil.bytes("c")));
1:a991b64:                 if (r != null)
1:a991b64:                 {
1:a991b64:                     if (r.getCell(withCommit) != null)
1:a991b64:                         commitLogMessageFound = true;
1:a991b64:                     if (r.getCell(withoutCommit) != null)
1:a991b64:                         noCommitLogMessageFound = true;
1:a991b64:                 }
1:de6dba7:             }
1:36958f3:             return true;
1:de6dba7:         }
1:de6dba7:     }
1:de6dba7: }
============================================================================
author:Robert Stupp
-------------------------------------------------------------------------------
commit:9797511
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.config.DatabaseDescriptor;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1:         DatabaseDescriptor.daemonInitialization();
author:Josh McKenzie
-------------------------------------------------------------------------------
commit:e31e216
/////////////////////////////////////////////////////////////////////////
0: import org.apache.cassandra.service.CassandraDaemon;
0: import org.apache.cassandra.service.StorageService;
/////////////////////////////////////////////////////////////////////////
0:         CassandraDaemon daemon = new CassandraDaemon();
0:         daemon.completeSetup(); //startup must be completed, otherwise commit log failure must kill JVM regardless of failure policy
0:         StorageService.instance.registerDaemon(daemon);
1: 
/////////////////////////////////////////////////////////////////////////
1: 
1:         CommitLogTestReplayer replayer = new CommitLogTestReplayer(checker);
1:         replayer.examineCommitLog();
author:Benedict Elliott Smith
-------------------------------------------------------------------------------
commit:2f41243
/////////////////////////////////////////////////////////////////////////
1:                 Row r = upd.getRow(Clustering.make(ByteBufferUtil.bytes("c")));
author:Ariel Weisberg
-------------------------------------------------------------------------------
commit:c4c9eae
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.io.util.DataInputBuffer;
/////////////////////////////////////////////////////////////////////////
1:         DataInputPlus dis = new DataInputBuffer(out.getData());
commit:03f72ac
/////////////////////////////////////////////////////////////////////////
1: 
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.io.util.DataInputPlus;
0: import org.apache.cassandra.io.util.NIODataInputStream;
/////////////////////////////////////////////////////////////////////////
0:         DataInputPlus dis = new NIODataInputStream(out.getData());
1:         return rms.deserialize(dis, MessagingService.current_version);
author:Sylvain Lebresne
-------------------------------------------------------------------------------
commit:2457599
/////////////////////////////////////////////////////////////////////////
0:                 Row r = upd.getRow(new Clustering(ByteBufferUtil.bytes("c")));
commit:a991b64
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.config.CFMetaData;
1: import org.apache.cassandra.config.ColumnDefinition;
0: import org.apache.cassandra.config.DatabaseDescriptor;
1: import org.apache.cassandra.db.partitions.PartitionUpdate;
1: import org.apache.cassandra.db.rows.Row;
0: import org.apache.cassandra.db.rows.RowIterator;
1: import org.apache.cassandra.db.marshal.AsciiType;
1: import org.apache.cassandra.db.marshal.BytesType;
0: import org.apache.cassandra.db.partitions.PartitionIterator;
1: import org.apache.cassandra.db.partitions.FilteredPartition;
1: import org.apache.cassandra.io.IVersionedSerializer;
1: import static org.junit.Assert.*;
1:     private static final String CF_FOR_READ_TEST = "Standard2";
1:     private static final String CF_FOR_COMMIT_TEST = "Standard3";
1:         CFMetaData cfForReadMetadata = CFMetaData.Builder.create(KEYSPACE1, CF_FOR_READ_TEST)
1:                                                             .addPartitionKey("key", BytesType.instance)
1:                                                             .addClusteringColumn("col1", AsciiType.instance)
1:                                                             .addClusteringColumn("col2", AsciiType.instance)
1:                                                             .addRegularColumn("a", AsciiType.instance)
1:                                                             .addRegularColumn("b", AsciiType.instance).build();
1: 
1:         CFMetaData cfForCommitMetadata1 = CFMetaData.Builder.create(KEYSPACE1, CF_FOR_COMMIT_TEST)
1:                                                        .addPartitionKey("key", BytesType.instance)
1:                                                        .addClusteringColumn("name", AsciiType.instance)
1:                                                        .addRegularColumn("commit1", AsciiType.instance).build();
1: 
1:         CFMetaData cfForCommitMetadata2 = CFMetaData.Builder.create(KEYSPACENOCOMMIT, CF_FOR_COMMIT_TEST)
1:                                                             .addPartitionKey("key", BytesType.instance)
1:                                                             .addClusteringColumn("name", AsciiType.instance)
1:                                                             .addRegularColumn("commit2", AsciiType.instance).build();
1: 
1:                                     SchemaLoader.standardCFMD(KEYSPACE1, CF),
1:                                     cfForReadMetadata,
1:                                     cfForCommitMetadata1);
1:                                     SchemaLoader.standardCFMD(KEYSPACENOCOMMIT, CF),
1:                                     cfForCommitMetadata2);
1:         ColumnFamilyStore cfs = Keyspace.open(KEYSPACE1).getColumnFamilyStore(CF_FOR_READ_TEST);
1:         rm = Util.cmd(cfs, Util.dk("key1"))
1:                  .includeRow("col1", "col2")
1:                  .build();
1:         assertEquals(rm.toString(), rm2.toString());
1:         rm = Util.cmd(cfs, Util.dk("key1"))
1:                  .includeRow("col1", "col2")
1:                  .reverse()
1:                  .build();
1:         assertEquals(rm.toString(), rm2.toString());
1:         rm = Util.cmd(cfs)
1:                  .build();
1:         assertEquals(rm.toString(), rm2.toString());
1: 
1:         rm = Util.cmd(cfs)
1:                  .fromKeyIncl(ByteBufferUtil.bytes("key1"))
1:                  .toKeyIncl(ByteBufferUtil.bytes("key2"))
1:                  .build();
1:         rm2 = serializeAndDeserializeReadMessage(rm);
1:         assertEquals(rm.toString(), rm2.toString());
1: 
1:         rm = Util.cmd(cfs)
1:                  .columns("a")
1:                  .build();
1:         rm2 = serializeAndDeserializeReadMessage(rm);
1:         assertEquals(rm.toString(), rm2.toString());
1: 
1:         rm = Util.cmd(cfs)
1:                  .includeRow("col1", "col2")
1:                  .columns("a")
1:                  .build();
1:         rm2 = serializeAndDeserializeReadMessage(rm);
1:         assertEquals(rm.toString(), rm2.toString());
1: 
1:         rm = Util.cmd(cfs)
1:                  .fromKeyIncl(ByteBufferUtil.bytes("key1"))
1:                  .includeRow("col1", "col2")
1:                  .columns("a")
1:                  .build();
1:         rm2 = serializeAndDeserializeReadMessage(rm);
1:         assertEquals(rm.toString(), rm2.toString());
1:         IVersionedSerializer<ReadCommand> rms = ReadCommand.serializer;
1: 
1: 
1:         ColumnFamilyStore cfs = Keyspace.open(KEYSPACE1).getColumnFamilyStore(CF);
1:         new RowUpdateBuilder(cfs.metadata, 0, ByteBufferUtil.bytes("key1"))
1:                 .clustering("Column1")
1:                 .add("val", ByteBufferUtil.bytes("abcd"))
1:                 .build()
1:                 .apply();
1:         ColumnDefinition col = cfs.metadata.getColumnDefinition(ByteBufferUtil.bytes("val"));
1:         int found = 0;
1:         for (FilteredPartition partition : Util.getAll(Util.cmd(cfs).build()))
1:         {
1:             for (Row r : partition)
1:             {
1:                 if (r.getCell(col).value().equals(ByteBufferUtil.bytes("abcd")))
1:                     ++found;
1:             }
1:         }
1:         assertEquals(1, found);
1:         ColumnFamilyStore cfs = Keyspace.open(KEYSPACE1).getColumnFamilyStore(CF_FOR_COMMIT_TEST);
1:         ColumnFamilyStore cfsnocommit = Keyspace.open(KEYSPACENOCOMMIT).getColumnFamilyStore(CF_FOR_COMMIT_TEST);
1:         new RowUpdateBuilder(cfs.metadata, 0, ByteBufferUtil.bytes("row"))
1:                 .clustering("c")
1:                 .add("commit1", ByteBufferUtil.bytes("abcd"))
1:                 .build()
1:                 .apply();
1: 
1:         new RowUpdateBuilder(cfsnocommit.metadata, 0, ByteBufferUtil.bytes("row"))
1:                 .clustering("c")
1:                 .add("commit2", ByteBufferUtil.bytes("abcd"))
1:                 .build()
1:                 .apply();
1: 
1:         Checker checker = new Checker(cfs.metadata.getColumnDefinition(ByteBufferUtil.bytes("commit1")),
1:                                       cfsnocommit.metadata.getColumnDefinition(ByteBufferUtil.bytes("commit2")));
/////////////////////////////////////////////////////////////////////////
1:         private final ColumnDefinition withCommit;
1:         private final ColumnDefinition withoutCommit;
1: 
1:         public Checker(ColumnDefinition withCommit, ColumnDefinition withoutCommit)
1:         {
1:             this.withCommit = withCommit;
1:             this.withoutCommit = withoutCommit;
1:         }
1: 
1:             for (PartitionUpdate upd : mutation.getPartitionUpdates())
0:                 Row r = upd.getRow(new SimpleClustering(ByteBufferUtil.bytes("c")));
1:                 if (r != null)
1:                 {
1:                     if (r.getCell(withCommit) != null)
1:                         commitLogMessageFound = true;
1:                     if (r.getCell(withoutCommit) != null)
1:                         noCommitLogMessageFound = true;
1:                 }
commit:e50d6af
/////////////////////////////////////////////////////////////////////////
0:         Cell col = row.cf.getColumn(Util.cellname("Column1"));
commit:362cc05
/////////////////////////////////////////////////////////////////////////
0: import org.apache.cassandra.db.composites.*;
0: import org.apache.cassandra.utils.FBUtilities;
/////////////////////////////////////////////////////////////////////////
0:         CellNameType type = Keyspace.open("Keyspace1").getColumnFamilyStore("Standard1").getComparator();
1: 
0:         SortedSet<CellName> colList = new TreeSet<CellName>(type);
0:         colList.add(Util.cellname("col1"));
0:         colList.add(Util.cellname("col2"));
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", ts, new SliceQueryFilter(Composites.EMPTY, Composites.EMPTY, true, 2));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", ts, new SliceQueryFilter(Util.cellname("a"), Util.cellname("z"), true, 5));
/////////////////////////////////////////////////////////////////////////
0:         CellNameType type = keyspace.getColumnFamilyStore("Standard1").getComparator();
0:         rm.add("Standard1", Util.cellname("Column1"), ByteBufferUtil.bytes("abcd"), 0);
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.key, "Standard1", System.currentTimeMillis(), new NamesQueryFilter(FBUtilities.singleton(Util.cellname("Column1"), type)));
0:         Column col = row.cf.getColumn(Util.cellname("Column1"));
0:         rm.add("Standard1", Util.cellname("commit1"), ByteBufferUtil.bytes("abcd"), 0);
0:         rm.add("Standard1", Util.cellname("commit2"), ByteBufferUtil.bytes("abcd"), 0);
commit:3a005df
/////////////////////////////////////////////////////////////////////////
0: import java.util.SortedSet;
0: import java.util.TreeSet;
0: import org.apache.cassandra.db.filter.NamesQueryFilter;
0: import org.apache.cassandra.db.filter.SliceQueryFilter;
/////////////////////////////////////////////////////////////////////////
0:         SortedSet<ByteBuffer> colList = new TreeSet<ByteBuffer>();
0:         rm = new SliceByNamesReadCommand("Keyspace1", dk.key, "Standard1", new NamesQueryFilter(colList));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", new SliceQueryFilter(ByteBufferUtil.EMPTY_BYTE_BUFFER, ByteBufferUtil.EMPTY_BYTE_BUFFER, true, 2));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", new SliceQueryFilter(ByteBufferUtil.bytes("a"), ByteBufferUtil.bytes("z"), true, 5));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", new SliceQueryFilter(ByteBufferUtil.EMPTY_BYTE_BUFFER, ByteBufferUtil.EMPTY_BYTE_BUFFER, true, 2));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", new SliceQueryFilter(ByteBufferUtil.bytes("a"), ByteBufferUtil.bytes("z"), true, 5));
/////////////////////////////////////////////////////////////////////////
0:         rm.add("Standard1", ByteBufferUtil.bytes("Column1"), ByteBufferUtil.bytes("abcd"), 0);
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.key, "Standard1", new NamesQueryFilter(ByteBufferUtil.bytes("Column1")));
0:         Column col = row.cf.getColumn(ByteBufferUtil.bytes("Column1"));
/////////////////////////////////////////////////////////////////////////
0:         rm.add("Standard1", ByteBufferUtil.bytes("commit1"), ByteBufferUtil.bytes("abcd"), 0);
0:         rm.add("Standard1", ByteBufferUtil.bytes("commit2"), ByteBufferUtil.bytes("abcd"), 0);
commit:07cdfd0
/////////////////////////////////////////////////////////////////////////
1: 
1: 
/////////////////////////////////////////////////////////////////////////
1: 
/////////////////////////////////////////////////////////////////////////
1: 
/////////////////////////////////////////////////////////////////////////
1: 
1:     @Test
1: 
1: 
1: 
1: 
1: 
1: 
1: 
1: 
1: 
0:         assertFalse(noCommitLogMessageFound);
1: 
commit:2fd3268
/////////////////////////////////////////////////////////////////////////
1: 
1: 
/////////////////////////////////////////////////////////////////////////
1: 
/////////////////////////////////////////////////////////////////////////
1: 
/////////////////////////////////////////////////////////////////////////
0: 
1:     @Test
0: 
0: 
0: 
0: 
0: 
0: 
0: 
0: 
0: 
0:         assertFalse(noCommitLogMessageFound);
0: 
commit:910b663
/////////////////////////////////////////////////////////////////////////
0:         rms.serialize(rm, dos, MessagingService.current_version);
0:         return rms.deserialize(new DataInputStream(bis), MessagingService.current_version);
author:Aleksey Yeschenko
-------------------------------------------------------------------------------
commit:31e3f61
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.schema.KeyspaceParams;
/////////////////////////////////////////////////////////////////////////
1:                                     KeyspaceParams.simple(1),
1:                                     KeyspaceParams.simpleTransient(1),
commit:6bbb13b
/////////////////////////////////////////////////////////////////////////
0:         Mutation rm;
0:         rm = new Mutation("Keyspace1", dk.key);
/////////////////////////////////////////////////////////////////////////
0:         Mutation rm = new Mutation("Keyspace1", ByteBufferUtil.bytes("row"));
0:         rm = new Mutation("NoCommitlogSpace", ByteBufferUtil.bytes("row"));
commit:0e96e58
/////////////////////////////////////////////////////////////////////////
0:         Keyspace keyspace = Keyspace.open("Keyspace1");
/////////////////////////////////////////////////////////////////////////
0:         Row row = command.getRow(keyspace);
commit:1f7628c
/////////////////////////////////////////////////////////////////////////
0:         long ts = System.currentTimeMillis();
0:         rm = new SliceByNamesReadCommand("Keyspace1", dk.key, "Standard1", ts, new NamesQueryFilter(colList));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", ts, new SliceQueryFilter(ByteBufferUtil.EMPTY_BYTE_BUFFER, ByteBufferUtil.EMPTY_BYTE_BUFFER, true, 2));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", ts, new SliceQueryFilter(ByteBufferUtil.bytes("a"), ByteBufferUtil.bytes("z"), true, 5));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", ts, new SliceQueryFilter(ByteBufferUtil.EMPTY_BYTE_BUFFER, ByteBufferUtil.EMPTY_BYTE_BUFFER, true, 2));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, "Standard1", ts, new SliceQueryFilter(ByteBufferUtil.bytes("a"), ByteBufferUtil.bytes("z"), true, 5));
/////////////////////////////////////////////////////////////////////////
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.key, "Standard1", System.currentTimeMillis(), new NamesQueryFilter(ByteBufferUtil.bytes("Column1")));
author:Branimir Lambov
-------------------------------------------------------------------------------
commit:36958f3
/////////////////////////////////////////////////////////////////////////
1: import com.google.common.base.Predicate;
1: import org.apache.cassandra.db.commitlog.CommitLogTestReplayer;
/////////////////////////////////////////////////////////////////////////
0:         Checker checker = new Checker();
0:         CommitLogTestReplayer.examineCommitLog(checker);
0: 
1:         assertTrue(checker.commitLogMessageFound);
1:         assertFalse(checker.noCommitLogMessageFound);
1:     }
0: 
1:     static class Checker implements Predicate<Mutation>
1:     {
1:         public boolean apply(Mutation mutation)
0:             for (ColumnFamily cf : mutation.getColumnFamilies())
0:                 if (cf.getColumn(Util.cellname("commit1")) != null)
0:                     commitLogMessageFound = true;
0:                 if (cf.getColumn(Util.cellname("commit2")) != null)
0:                     noCommitLogMessageFound = true;
1:             return true;
author:lyubent
-------------------------------------------------------------------------------
commit:d2a3827
/////////////////////////////////////////////////////////////////////////
1: import org.junit.BeforeClass;
0: import org.apache.cassandra.config.KSMetaData;
1: import org.apache.cassandra.exceptions.ConfigurationException;
0: import org.apache.cassandra.locator.SimpleStrategy;
1: public class ReadMessageTest
1:     private static final String KEYSPACE1 = "ReadMessageTest1";
1:     private static final String KEYSPACENOCOMMIT = "ReadMessageTest_NoCommit";
1:     private static final String CF = "Standard1";
0: 
1:     @BeforeClass
1:     public static void defineSchema() throws ConfigurationException
1:     {
1:         SchemaLoader.prepareServer();
1:         SchemaLoader.createKeyspace(KEYSPACE1,
0:                                     SimpleStrategy.class,
0:                                     KSMetaData.optsWithRF(1),
0:                                     SchemaLoader.standardCFMD(KEYSPACE1, CF));
1:         SchemaLoader.createKeyspace(KEYSPACENOCOMMIT,
0:                                     false,
0:                                     true,
0:                                     SimpleStrategy.class,
0:                                     KSMetaData.optsWithRF(1),
0:                                     SchemaLoader.standardCFMD(KEYSPACENOCOMMIT, CF));
1:     }
0: 
0:         CellNameType type = Keyspace.open(KEYSPACE1).getColumnFamilyStore("Standard1").getComparator();
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceByNamesReadCommand(KEYSPACE1, dk.getKey(), "Standard1", ts, new NamesQueryFilter(colList));
0:         rm = new SliceFromReadCommand(KEYSPACE1, dk.getKey(), "Standard1", ts, new SliceQueryFilter(Composites.EMPTY, Composites.EMPTY, true, 2));
0:         rm = new SliceFromReadCommand(KEYSPACE1, dk.getKey(), "Standard1", ts, new SliceQueryFilter(Util.cellname("a"), Util.cellname("z"), true, 5));
/////////////////////////////////////////////////////////////////////////
0:         Keyspace keyspace = Keyspace.open(KEYSPACE1);
0:         rm = new Mutation(KEYSPACE1, dk.getKey());
0:         ReadCommand command = new SliceByNamesReadCommand(KEYSPACE1, dk.getKey(), "Standard1", System.currentTimeMillis(), new NamesQueryFilter(FBUtilities.singleton(Util.cellname("Column1"), type)));
/////////////////////////////////////////////////////////////////////////
0:         Mutation rm = new Mutation(KEYSPACE1, ByteBufferUtil.bytes("row"));
0:         rm = new Mutation(KEYSPACENOCOMMIT, ByteBufferUtil.bytes("row"));
author:Pavel Yaskevich
-------------------------------------------------------------------------------
commit:8541cca
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceByNamesReadCommand("Keyspace1", dk.getKey(), "Standard1", ts, new NamesQueryFilter(colList));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.getKey(), "Standard1", ts, new SliceQueryFilter(Composites.EMPTY, Composites.EMPTY, true, 2));
0:         rm = new SliceFromReadCommand("Keyspace1", dk.getKey(), "Standard1", ts, new SliceQueryFilter(Util.cellname("a"), Util.cellname("z"), true, 5));
/////////////////////////////////////////////////////////////////////////
0:         rm = new Mutation("Keyspace1", dk.getKey());
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.getKey(), "Standard1", System.currentTimeMillis(), new NamesQueryFilter(FBUtilities.singleton(Util.cellname("Column1"), type)));
author:Dave Brosius
-------------------------------------------------------------------------------
commit:9639f95
/////////////////////////////////////////////////////////////////////////
1:     public void testGetColumn()
commit:b5c23cf
commit:00e871d
/////////////////////////////////////////////////////////////////////////
0:         DataInputStream dis = new DataInputStream(is);
0:         dis.mark(100);
0:         dis.readFully(lookahead);
0:         dis.reset();
commit:bc6b5f4
commit:de6dba7
/////////////////////////////////////////////////////////////////////////
0:         byte[] commitBytes = "commit".getBytes("UTF-8");
0: 
0:             BufferedInputStream is = null;
0:             try
0:                 is = new BufferedInputStream(new FileInputStream(commitLogDir.getAbsolutePath()+File.separator+filename));
0:                 if (!isEmptyCommitLog(is))
1:                 {
0:                     while (findPatternInStream(commitBytes, is))
1:                     {
0:                         char c = (char)is.read();
0: 
0:                         if (c == '1')
0:                             commitLogMessageFound = true;
0:                         else if (c == '2')
0:                             noCommitLogMessageFound = true;
1:                     }
1:                 }
0:             finally
1:             {
0:                 if (is != null)
0:                     is.close();
1:             }
0:     private boolean isEmptyCommitLog(BufferedInputStream is) throws IOException
1:     {
0:         byte[] lookahead = new byte[100];
0: 
0:         is.mark(100);
0:         is.read(lookahead);
0:         is.reset();
0: 
0:         for (int i = 0; i < 100; i++)
1:         {
0:             if (lookahead[i] != 0)
0:                 return false;
1:         }
0: 
0:         return true;
1:     }
0: 
0:     private boolean findPatternInStream(byte[] pattern, InputStream is) throws IOException
1:     {
0:         int patternOffset = 0;
0: 
0:         int b = is.read();
0:         while (b != -1)
1:         {
0:             if (pattern[patternOffset] == ((byte) b))
1:             {
0:                 patternOffset++;
0:                 if (patternOffset == pattern.length)
0:                     return true;
1:             }
0:             else
0:                 patternOffset = 0;
0: 
0:             b = is.read();
1:         }
0: 
0:         return false;
1:     }
author:Jonathan Ellis
-------------------------------------------------------------------------------
commit:60d9c7f
/////////////////////////////////////////////////////////////////////////
1:         DataOutputBuffer out = new DataOutputBuffer();
1:         rms.serialize(rm, out, MessagingService.current_version);
0:         bis = new ByteArrayInputStream(out.getData(), 0, out.getLength());
commit:b95a49c
/////////////////////////////////////////////////////////////////////////
0:         assertEquals(col.value(), ByteBuffer.wrap("abcd".getBytes()));
commit:43d330d
/////////////////////////////////////////////////////////////////////////
0: 
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ByteBufferUtil.EMPTY_BYTE_BUFFER, ByteBufferUtil.EMPTY_BYTE_BUFFER, true, 2);
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ByteBufferUtil.EMPTY_BYTE_BUFFER, ByteBufferUtil.EMPTY_BYTE_BUFFER, true, 2);
commit:8358669
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.utils.ByteBufferUtil;
0: 
/////////////////////////////////////////////////////////////////////////
0:         colList.add(ByteBufferUtil.bytes("col1"));
0:         colList.add(ByteBufferUtil.bytes("col2"));
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ByteBufferUtil.bytes("a"), ByteBufferUtil.bytes("z"), true, 5);
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ByteBufferUtil.bytes("a"), ByteBufferUtil.bytes("z"), true, 5);
/////////////////////////////////////////////////////////////////////////
0:         rm.add(new QueryPath("Standard1", null, ByteBufferUtil.bytes("Column1")), ByteBufferUtil.bytes("abcd"), 0);
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), Arrays.asList(ByteBufferUtil.bytes("Column1")));
0:         IColumn col = row.cf.getColumn(ByteBufferUtil.bytes("Column1"));
commit:e7a385a
/////////////////////////////////////////////////////////////////////////
0: import static org.junit.Assert.assertEquals;
0: import java.nio.ByteBuffer;
1: import org.apache.cassandra.Util;
0: import org.apache.cassandra.utils.FBUtilities;
0: import org.apache.commons.lang.ArrayUtils;
1: import org.junit.Test;
0:         ArrayList<ByteBuffer> colList = new ArrayList<ByteBuffer>();
0:         colList.add(ByteBuffer.wrap("col1".getBytes()));
0:         colList.add(ByteBuffer.wrap("col2".getBytes()));
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"),FBUtilities.EMPTY_BYTE_BUFFER, FBUtilities.EMPTY_BYTE_BUFFER, true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ByteBuffer.wrap("a".getBytes()), ByteBuffer.wrap("z".getBytes()), true, 5);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), FBUtilities.EMPTY_BYTE_BUFFER, FBUtilities.EMPTY_BYTE_BUFFER, true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ByteBuffer.wrap("a".getBytes()), ByteBuffer.wrap("z".getBytes()), true, 5);
/////////////////////////////////////////////////////////////////////////
0:         rm.add(new QueryPath("Standard1", null, ByteBuffer.wrap("Column1".getBytes())), ByteBuffer.wrap("abcd".getBytes()), 0);
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), Arrays.asList(ByteBuffer.wrap("Column1".getBytes())));
0:         IColumn col = row.cf.getColumn(ByteBuffer.wrap("Column1".getBytes()));
0:         assert Arrays.equals(col.value().array(), "abcd".getBytes());  
commit:9d32382
/////////////////////////////////////////////////////////////////////////
0:         rm.add(new QueryPath("Standard1", null, "Column1".getBytes()), "abcd".getBytes(), 0);
commit:b2a8d89
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), true, 5);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), true, 5);
commit:cba59a8
/////////////////////////////////////////////////////////////////////////
0:         rm.add(new QueryPath("Standard1", null, "Column1".getBytes()), "abcd".getBytes(), new TimestampClock(0));
commit:7d2115b
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, new ArrayList<byte[]>(0), true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), new ArrayList<byte[]>(0), true, 5);
0: 
0:         rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, null, true, 2);
1:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assert rm2.toString().equals(rm.toString());
0: 
0:         rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), null, true, 5);
1:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assertEquals(rm2.toString(), rm.toString());
0: 
0:         for (String[] bitmaskTests: new String[][] { {}, {"test one", "test two" }, { new String(new byte[] { 0, 1, 0x20, (byte) 0xff }) } })
0:         {
0:             ArrayList<byte[]> bitmasks = new ArrayList<byte[]>(bitmaskTests.length);
0:             for (String bitmaskTest : bitmaskTests)
0:             {
0:                 bitmasks.add(bitmaskTest.getBytes("UTF-8"));
0:             }
0: 
0:             rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, bitmasks, true, 2);
1:             rm2 = serializeAndDeserializeReadMessage(rm);
0:             assert rm2.toString().equals(rm.toString());
0: 
0:             rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), bitmasks, true, 5);
1:             rm2 = serializeAndDeserializeReadMessage(rm);
0:             assertEquals(rm2.toString(), rm.toString());
0:         }
commit:3655e91
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.io.util.DataOutputBuffer;
commit:dc6e4fe
/////////////////////////////////////////////////////////////////////////
commit:1cb0794
/////////////////////////////////////////////////////////////////////////
0: import java.io.ByteArrayInputStream;
0: import java.io.DataInputStream;
/////////////////////////////////////////////////////////////////////////
/////////////////////////////////////////////////////////////////////////
0:         ByteArrayInputStream bis;
0:         bis = new ByteArrayInputStream(dos.getData(), 0, dos.getLength());
0:         return rms.deserialize(new DataInputStream(bis));
commit:994487d
/////////////////////////////////////////////////////////////////////////
0:         IColumn col = row.cf.getColumn("Column1".getBytes());
commit:572b5f8
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceByNamesReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), colList);
0:         rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", "row1", new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), true, 5);
/////////////////////////////////////////////////////////////////////////
0:         Table table = Table.open("Keyspace1");
0:         rm = new RowMutation("Keyspace1", "key1");
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", "key1", new QueryPath("Standard1"), Arrays.asList("Column1".getBytes()));
commit:986cee6
/////////////////////////////////////////////////////////////////////////
0: import org.apache.commons.lang.ArrayUtils;
0: import org.apache.cassandra.db.marshal.AsciiType;
1:     public void testMakeReadMessage() throws IOException
0:         ArrayList<byte[]> colList = new ArrayList<byte[]>();
0:         colList.add("col1".getBytes());
0:         colList.add("col2".getBytes());
0:         rm = new SliceByNamesReadCommand("Table1", "row1", new QueryPath("Standard1"), colList);
0:         rm = new SliceFromReadCommand("Table1", "row1", new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, true, 2);
0:         rm = new SliceFromReadCommand("Table1", "row1", new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), true, 5);
1:     private ReadCommand serializeAndDeserializeReadMessage(ReadCommand rm) throws IOException
0:         rms.serialize(rm, dos);
0:         dis.reset(dos.getData(), dos.getLength());
0:         return rms.deserialize(dis);
/////////////////////////////////////////////////////////////////////////
0:         rm.add(new QueryPath("Standard1", null, "Column1".getBytes()), "abcd".getBytes(), 0);
0:         ReadCommand command = new SliceByNamesReadCommand("Table1", "key1", new QueryPath("Standard1"), Arrays.asList("Column1".getBytes()));
0:         IColumn col = cf.getColumn("Column1".getBytes());
0:         assert Arrays.equals(col.value(), "abcd".getBytes());  
commit:8ff63a9
/////////////////////////////////////////////////////////////////////////
commit:ff764ff
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Table1", "row1", new QueryPath("foo"), "", "", true, 2);
0:         rm = new SliceFromReadCommand("Table1", "row1", new QueryPath("foo"), "a", "z", true, 5);
commit:f2da00f
/////////////////////////////////////////////////////////////////////////
0: import org.apache.cassandra.db.filter.QueryPath;
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceByNamesReadCommand("Table1", "row1", new QueryPath("foo"), colList);
0:         rm = new ColumnsSinceReadCommand("Table1", "row1", new QueryPath("foo"), 1);
0:         rm = new SliceFromReadCommand("Table1", "row1", new QueryPath("foo"), "", "", true, 0, 2);
0:         rm = new SliceFromReadCommand("Table1", "row1", new QueryPath("foo"), "a", "z", true, 0, 5);
/////////////////////////////////////////////////////////////////////////
0:         rm.add(new QueryPath("Standard1", null, "Column1"), "abcd".getBytes(), 0);
0:         ReadCommand command = new SliceByNamesReadCommand("Table1", "key1", new QueryPath("Standard1"), Arrays.asList("Column1"));
commit:f5a787a
/////////////////////////////////////////////////////////////////////////
commit:23fa1bc
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Table1", "row1", "foo", "", "", true, 0, 2);
0:         rm = new SliceFromReadCommand("Table1", "row1", "foo", "a", "z", true, 0, 5);
commit:cdd8b17
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Table1", "row1", "foo", true, 0, 2);
commit:6342512
/////////////////////////////////////////////////////////////////////////
0:         rm = new SliceFromReadCommand("Table1", "row1", "foo", true, 2);
commit:88edbed
/////////////////////////////////////////////////////////////////////////
1: /*
1: * Licensed to the Apache Software Foundation (ASF) under one
1: * or more contributor license agreements.  See the NOTICE file
1: * distributed with this work for additional information
1: * regarding copyright ownership.  The ASF licenses this file
1: * to you under the Apache License, Version 2.0 (the
1: * "License"); you may not use this file except in compliance
1: * with the License.  You may obtain a copy of the License at
1: *
1: *    http://www.apache.org/licenses/LICENSE-2.0
1: *
1: * Unless required by applicable law or agreed to in writing,
1: * software distributed under the License is distributed on an
1: * "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
1: * KIND, either express or implied.  See the License for the
1: * specific language governing permissions and limitations
1: * under the License.
1: */
commit:20f7d03
/////////////////////////////////////////////////////////////////////////
0: import static org.junit.Assert.*;
0: 
0: import org.junit.Assert;
/////////////////////////////////////////////////////////////////////////
0:         
0:         rm = new SliceByRangeReadCommand("Table1", "row1", "foo", "a", "z", 5);
0:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assertEquals(rm2.toString(), rm.toString());
commit:97fc5cb
/////////////////////////////////////////////////////////////////////////
0: import org.junit.Test;
0: 
commit:afd3c27
commit:e2a4ea7
/////////////////////////////////////////////////////////////////////////
0:         
1:         ReadCommand rm, rm2;
0:         
0:         rm = new SliceByNamesReadCommand("Table1", "row1", "foo", colList);
0:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assert rm2.toString().equals(rm.toString());
0:         rm = new ColumnReadCommand("Table1", "row1", "foo:col1");
0:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assert rm2.toString().equals(rm.toString());
0:         rm = new RowReadCommand("Table1", "row1");
0:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assert rm2.toString().equals(rm.toString());
0: 
0:         rm = new ColumnsSinceReadCommand("Table1", "row1", "foo", 1);
0:         rm2 = serializeAndDeserializeReadMessage(rm);
0:         assert rm2.toString().equals(rm.toString());
0: 
0:         rm = new SliceReadCommand("Table1", "row1", "foo", 1, 2);
0:         rm2 = serializeAndDeserializeReadMessage(rm);
/////////////////////////////////////////////////////////////////////////
0:         ReadCommand command = new ColumnReadCommand("Table1", "key1", "Standard1:Column1");
commit:d14f814
/////////////////////////////////////////////////////////////////////////
0: import static org.testng.Assert.assertNull;
0: 
0: import java.util.Arrays;
/////////////////////////////////////////////////////////////////////////
0:     
1:     @Test
0:     public void testGetColumn() throws IOException, ColumnFamilyNotDefinedException
0:     {
0:         Table table = Table.open("Table1");
0:         RowMutation rm;
0: 
0:         // add data
0:         rm = new RowMutation("Table1", "key1");
0:         rm.add("Standard1:Column1", "abcd".getBytes(), 0);
0:         rm.apply();
0: 
0:         ReadCommand command = new ReadCommand("Table1", "key1", "Standard1:Column1", -1, Integer.MAX_VALUE);
0:         Row row = command.getRow(table);
0:         ColumnFamily cf = row.getColumnFamily("Standard1");
0:         IColumn col = cf.getColumn("Column1");
0:         assert Arrays.equals(((Column)col).value(), "abcd".getBytes());  
0:     }
commit:df95fb8
/////////////////////////////////////////////////////////////////////////
0:         ReadCommandSerializer rms = ReadCommand.serializer();
commit:8e72ac4
/////////////////////////////////////////////////////////////////////////
0:         ReadCommand rm = new ReadCommand("Table1", "row1", "foo", colList);
0:         ReadCommand rm2 = serializeAndDeserializeReadMessage(rm);
0:     private ReadCommand serializeAndDeserializeReadMessage(ReadCommand rm)
0:         ReadCommand rm2 = null;
0:         ReadCommandSerializer rms = (ReadCommandSerializer) ReadCommand.serializer();
commit:6931092
/////////////////////////////////////////////////////////////////////////
1: package org.apache.cassandra.db;
0: 
0: import java.io.IOException;
0: import java.util.ArrayList;
0: 
0: import org.apache.cassandra.io.DataInputBuffer;
0: import org.apache.cassandra.io.DataOutputBuffer;
0: import org.testng.annotations.Test;
0: 
0: public class ReadMessageTest
0: {
0:     @Test
0:     public void testMakeReadMessage()
0:     {
0:         ArrayList<String> colList = new ArrayList<String>();
0:         colList.add("col1");
0:         colList.add("col2");
0: 
0:         ReadMessage rm = new ReadMessage("Table1", "row1", "foo", colList);
0:         ReadMessage rm2 = serializeAndDeserializeReadMessage(rm);
0: 
0:         assert rm2.toString().equals(rm.toString());
0:     }
0: 
0:     private ReadMessage serializeAndDeserializeReadMessage(ReadMessage rm)
0:     {
0:         ReadMessage rm2 = null;
0:         ReadMessageSerializer rms = (ReadMessageSerializer) ReadMessage.serializer();
0:         DataOutputBuffer dos = new DataOutputBuffer();
0:         DataInputBuffer dis = new DataInputBuffer();
0: 
0:         try
0:         {
0:             rms.serialize(rm, dos);
0:             dis.reset(dos.getData(), dos.getLength());
0:             rm2 = rms.deserialize(dis);
0:         }
0:         catch (IOException e)
0:         {
0:             throw new RuntimeException(e);
0:         }
0:         return rm2;
0:     }
0: }
author:Yuki Morishita
-------------------------------------------------------------------------------
commit:587cb58
/////////////////////////////////////////////////////////////////////////
0:         ReadCommandSerializer rms = ReadCommand.serializer;
author:T Jake Luciani
-------------------------------------------------------------------------------
commit:ea6ec42
/////////////////////////////////////////////////////////////////////////
0: import static org.junit.Assert.*;
1: import java.io.*;
0: import org.junit.Test;
0: 
0: import org.apache.cassandra.config.DatabaseDescriptor;
/////////////////////////////////////////////////////////////////////////
0:     
0:     @Test 
1:     public void testNoCommitLog() throws Exception
0:     {
0:                    
0:         RowMutation rm = new RowMutation("Keyspace1", ByteBufferUtil.bytes("row"));
0:         rm.add(new QueryPath("Standard1", null, ByteBufferUtil.bytes("commit1")), ByteBufferUtil.bytes("abcd"), 0);
0:         rm.apply();
0:           
0:         rm = new RowMutation("NoCommitlogSpace", ByteBufferUtil.bytes("row"));
0:         rm.add(new QueryPath("Standard1", null, ByteBufferUtil.bytes("commit2")), ByteBufferUtil.bytes("abcd"), 0);
0:         rm.apply();
0:         
1:         boolean commitLogMessageFound = false;
1:         boolean noCommitLogMessageFound = false;
0:             
0:         File commitLogDir = new File(DatabaseDescriptor.getCommitLogLocation());
0:             
0:         for(String filename : commitLogDir.list())
0:         {
0:             BufferedReader f = new BufferedReader(new FileReader(commitLogDir.getAbsolutePath()+File.separator+filename));
0:                 
0:             String line = null;
0:             while( (line = f.readLine()) != null)
0:             {
0:                 if(line.contains("commit1"))
0:                     commitLogMessageFound = true;
0:                     
0:                 if(line.contains("commit2"))
0:                     noCommitLogMessageFound = true;
0:             }
0:                 
0:             f.close();
0:         }
0:             
0:         assertTrue(commitLogMessageFound);
0:         assertFalse(noCommitLogMessageFound);         
0:     }
0:     
author:Gary Dusbabek
-------------------------------------------------------------------------------
commit:1ecdd7f
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.net.MessagingService;
/////////////////////////////////////////////////////////////////////////
0:         rms.serialize(rm, dos, MessagingService.version_);
0:         return rms.deserialize(new DataInputStream(bis), MessagingService.version_);
commit:434564d
/////////////////////////////////////////////////////////////////////////
0: import org.apache.cassandra.Util;
0: 
/////////////////////////////////////////////////////////////////////////
0:         DecoratedKey dk = Util.dk("row1");
0:         rm = new SliceByNamesReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), colList);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, new ArrayList<byte[]>(0), true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), new ArrayList<byte[]>(0), true, 5);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, null, true, 2);
0:         rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), null, true, 5);
/////////////////////////////////////////////////////////////////////////
0:             rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), ArrayUtils.EMPTY_BYTE_ARRAY, ArrayUtils.EMPTY_BYTE_ARRAY, bitmasks, true, 2);
0:             rm = new SliceFromReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), "a".getBytes(), "z".getBytes(), bitmasks, true, 5);
/////////////////////////////////////////////////////////////////////////
0:         DecoratedKey dk = Util.dk("key1");
0:         rm = new RowMutation("Keyspace1", dk.key);
0:         ReadCommand command = new SliceByNamesReadCommand("Keyspace1", dk.key, new QueryPath("Standard1"), Arrays.asList("Column1".getBytes()));
commit:7a75b63
/////////////////////////////////////////////////////////////////////////
1: import org.apache.cassandra.SchemaLoader;
0: public class ReadMessageTest extends SchemaLoader
============================================================================
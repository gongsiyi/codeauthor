1:0dfad31: /*
4:0dfad31: 
1:0dfad31:    Derby - Class org.apache.derbyTesting.functionTests.tests.engine.ReadMeFilesTest
1:0dfad31: 
1:0dfad31: 
1:0dfad31:    Licensed to the Apache Software Foundation (ASF) under one or more
1:0dfad31:    contributor license agreements.  See the NOTICE file distributed with
1:0dfad31:    this work for additional information regarding copyright ownership.
1:0dfad31:    The ASF licenses this file to You under the Apache License, Version 2.0
1:0dfad31:    (the "License"); you may not use this file except in compliance with
1:0dfad31:    the License.  You may obtain a copy of the License at
1:0dfad31: 
1:0dfad31:       http://www.apache.org/licenses/LICENSE-2.0
1:0dfad31: 
1:0dfad31:    Unless required by applicable law or agreed to in writing, software
1:0dfad31:    distributed under the License is distributed on an "AS IS" BASIS,
1:0dfad31:    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:0dfad31:    See the License for the specific language governing permissions and
1:0dfad31:    limitations under the License.
1:0dfad31: 
1:0dfad31:  */
1:0dfad31: 
1:0dfad31: package org.apache.derbyTesting.functionTests.tests.engine;
1:0dfad31: 
1:0dfad31: import java.io.File;
1:0dfad31: import java.io.IOException;
1:0dfad31: import java.sql.SQLException;
1:0dfad31: import junit.framework.Test;
1:0dfad31: import org.apache.derbyTesting.functionTests.util.PrivilegedFileOpsForTests;
1:0dfad31: import org.apache.derbyTesting.junit.BaseJDBCTestCase;
1:b79d9d8: import org.apache.derbyTesting.junit.BaseTestCase;
1:1ae02c9: import org.apache.derbyTesting.junit.BaseTestSuite;
1:b79d9d8: import org.apache.derbyTesting.junit.Decorator;
1:0dfad31: import org.apache.derbyTesting.junit.TestConfiguration;
1:0dfad31: 
1:0dfad31: 
1:0dfad31: /**
1:0dfad31:  * Tests related to the 3 Derby readme files. These readmes warn users against 
1:0dfad31:  *   editing/deleting any of the files in the database directories. The 
1:0dfad31:  *   location of the readme files are  
1:0dfad31:  *   1)at the db level directory, 
1:0dfad31:  *   2)in seg0 directory and 
1:0dfad31:  *   3)in the log directocy.
1:70ddc11:  * All the three readme files are named README_DO_NOT_TOUCH_FILES.txt
1:0dfad31:  */
1:0dfad31: public class ReadMeFilesTest extends BaseJDBCTestCase {
1:0dfad31:     /**
1:0dfad31:     The readme file cautioning users against touching the files in
1:0dfad31:     the database directory 
1:0dfad31:     */
1:70ddc11:     private static final String DB_README_FILE_NAME = "README_DO_NOT_TOUCH_FILES.txt";
1:b79d9d8:     static String logDir = BaseTestCase.getSystemProperty("derby.system.home")+File.separator+"abcs";
1:0dfad31: 
1:0dfad31:     public ReadMeFilesTest(String name) {
1:0dfad31:         super(name);
1:0dfad31:     }
1:0dfad31: 
1:0dfad31:     public static Test suite() {
1:1ae02c9:         BaseTestSuite suite = new BaseTestSuite("ReadMeFilesTest");
1:b79d9d8: 
1:b79d9d8:         //DERBY-5232 (Put a stern README file in log and seg0 directories 
1:b79d9d8:         // to warn users of corrpution they will cause if they touch files 
1:b79d9d8:         // there)
1:b79d9d8:         //Test the existence of readme files for a default embedded config
1:b79d9d8:         // which means that "log" directory is under the database directory
1:b79d9d8:         // along with "seg0" directory
1:b79d9d8:         suite.addTest(TestConfiguration.singleUseDatabaseDecorator(
1:b79d9d8:             TestConfiguration.embeddedSuite(ReadMeFilesTest.class)));
1:b79d9d8: 
1:b79d9d8:         //DERBY-5995 (Add a test case to check the 3 readme files get created 
1:b79d9d8:         // even when log directory has been changed with jdbc url attribute 
1:b79d9d8:         // logDevice )
1:b79d9d8:         //Test the existence of readme files for a database configuration
1:b79d9d8:         // where "log" directory may not be under the database directory.
1:b79d9d8:         // It's location is determined by jdbc url attribute logDevice.
1:b79d9d8:         logDir = BaseTestCase.getSystemProperty("derby.system.home")+
1:b79d9d8:             File.separator+"abcs";
1:b79d9d8:         suite.addTest(
1:b79d9d8:             Decorator.logDeviceAttributeDatabase(
1:b79d9d8:                 TestConfiguration.embeddedSuite(ReadMeFilesTest.class),
1:b79d9d8:                 logDir));
1:b79d9d8:         return suite;
1:0dfad31:     }
1:0dfad31: 
1:0dfad31:     public void testReadMeFilesExist() throws IOException, SQLException {
1:0dfad31:         getConnection();
1:0dfad31:         TestConfiguration currentConfig = TestConfiguration.getCurrent();
1:0dfad31:         String dbPath = currentConfig.getDatabasePath(currentConfig.getDefaultDatabaseName());
1:0dfad31:         lookForReadmeFile(dbPath);
1:0dfad31:         lookForReadmeFile(dbPath+File.separator+"seg0");
1:b79d9d8: 
1:b79d9d8:         String logDevice = currentConfig.getConnectionAttributes().getProperty("logDevice");
1:b79d9d8:         if (logDevice != null) {
1:b79d9d8:             lookForReadmeFile(logDir+File.separator+"log");
1:b79d9d8:         } else {
1:b79d9d8:             lookForReadmeFile(dbPath+File.separator+"log");
1:b79d9d8:         }
1:0dfad31:     }
1:0dfad31: 
1:0dfad31:     private void lookForReadmeFile(String path) throws IOException {
1:0dfad31:         File readmeFile = new File(path,
1:0dfad31:             DB_README_FILE_NAME);
1:b79d9d8:         assertTrue(readmeFile + "doesn't exist", 
1:b79d9d8:             PrivilegedFileOpsForTests.exists(readmeFile));
1:0dfad31:     }
1:0dfad31: }
============================================================================
author:Dag H. Wanvik
-------------------------------------------------------------------------------
commit:1ae02c9
/////////////////////////////////////////////////////////////////////////
1: import org.apache.derbyTesting.junit.BaseTestSuite;
/////////////////////////////////////////////////////////////////////////
1:         BaseTestSuite suite = new BaseTestSuite("ReadMeFilesTest");
author:Mamta Satoor
-------------------------------------------------------------------------------
commit:b79d9d8
/////////////////////////////////////////////////////////////////////////
0: import junit.framework.TestSuite;
1: import org.apache.derbyTesting.junit.BaseTestCase;
1: import org.apache.derbyTesting.junit.Decorator;
/////////////////////////////////////////////////////////////////////////
1:     static String logDir = BaseTestCase.getSystemProperty("derby.system.home")+File.separator+"abcs";
0:     	TestSuite suite = new TestSuite("ReadMeFilesTest");
1: 
1:         //DERBY-5232 (Put a stern README file in log and seg0 directories 
1:         // to warn users of corrpution they will cause if they touch files 
1:         // there)
1:         //Test the existence of readme files for a default embedded config
1:         // which means that "log" directory is under the database directory
1:         // along with "seg0" directory
1:         suite.addTest(TestConfiguration.singleUseDatabaseDecorator(
1:             TestConfiguration.embeddedSuite(ReadMeFilesTest.class)));
1: 
1:         //DERBY-5995 (Add a test case to check the 3 readme files get created 
1:         // even when log directory has been changed with jdbc url attribute 
1:         // logDevice )
1:         //Test the existence of readme files for a database configuration
1:         // where "log" directory may not be under the database directory.
1:         // It's location is determined by jdbc url attribute logDevice.
1:         logDir = BaseTestCase.getSystemProperty("derby.system.home")+
1:             File.separator+"abcs";
1:         suite.addTest(
1:             Decorator.logDeviceAttributeDatabase(
1:                 TestConfiguration.embeddedSuite(ReadMeFilesTest.class),
1:                 logDir));
1:         return suite;
/////////////////////////////////////////////////////////////////////////
1: 
1:         String logDevice = currentConfig.getConnectionAttributes().getProperty("logDevice");
1:         if (logDevice != null) {
1:             lookForReadmeFile(logDir+File.separator+"log");
1:         } else {
1:             lookForReadmeFile(dbPath+File.separator+"log");
1:         }
1:         assertTrue(readmeFile + "doesn't exist", 
1:             PrivilegedFileOpsForTests.exists(readmeFile));
commit:70ddc11
/////////////////////////////////////////////////////////////////////////
1:  * All the three readme files are named README_DO_NOT_TOUCH_FILES.txt
1:     private static final String DB_README_FILE_NAME = "README_DO_NOT_TOUCH_FILES.txt";
commit:ce0c0ae
/////////////////////////////////////////////////////////////////////////
0:         assertTrue(readmeFile + "doesn't exist", PrivilegedFileOpsForTests.exists(readmeFile));
commit:0dfad31
/////////////////////////////////////////////////////////////////////////
1: /*
1: 
1:    Derby - Class org.apache.derbyTesting.functionTests.tests.engine.ReadMeFilesTest
1: 
1: 
1:    Licensed to the Apache Software Foundation (ASF) under one or more
1:    contributor license agreements.  See the NOTICE file distributed with
1:    this work for additional information regarding copyright ownership.
1:    The ASF licenses this file to You under the Apache License, Version 2.0
1:    (the "License"); you may not use this file except in compliance with
1:    the License.  You may obtain a copy of the License at
1: 
1:       http://www.apache.org/licenses/LICENSE-2.0
1: 
1:    Unless required by applicable law or agreed to in writing, software
1:    distributed under the License is distributed on an "AS IS" BASIS,
1:    WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
1:    See the License for the specific language governing permissions and
1:    limitations under the License.
1: 
1:  */
1: 
1: package org.apache.derbyTesting.functionTests.tests.engine;
1: 
1: import java.io.File;
1: import java.io.IOException;
1: import java.sql.SQLException;
1: 
1: import junit.framework.Test;
1: 
1: import org.apache.derbyTesting.functionTests.util.PrivilegedFileOpsForTests;
1: import org.apache.derbyTesting.junit.BaseJDBCTestCase;
1: import org.apache.derbyTesting.junit.TestConfiguration;
1: 
1: 
1: /**
1:  * Tests related to the 3 Derby readme files. These readmes warn users against 
1:  *   editing/deleting any of the files in the database directories. The 
1:  *   location of the readme files are  
1:  *   1)at the db level directory, 
1:  *   2)in seg0 directory and 
1:  *   3)in the log directocy.
0:  * All the three readme files are named README_DONT_TOUCH_FILES.txt
1:  */
1: public class ReadMeFilesTest extends BaseJDBCTestCase {
1:     /**
1:     The readme file cautioning users against touching the files in
1:     the database directory 
1:     */
0:     private static final String DB_README_FILE_NAME = "README_DONT_TOUCH_FILES.txt";
1: 
1:     public ReadMeFilesTest(String name) {
1:         super(name);
1:     }
1: 
1:     public static Test suite() {
0:         Test suite = TestConfiguration.embeddedSuite(ReadMeFilesTest.class);
0:         return TestConfiguration.singleUseDatabaseDecorator(suite);
1:     }
1: 
1:     public void testReadMeFilesExist() throws IOException, SQLException {
1:         getConnection();
1:         TestConfiguration currentConfig = TestConfiguration.getCurrent();
1:         String dbPath = currentConfig.getDatabasePath(currentConfig.getDefaultDatabaseName());
1:         lookForReadmeFile(dbPath);
1:         lookForReadmeFile(dbPath+File.separator+"seg0");
0:         lookForReadmeFile(dbPath+File.separator+"log");
1:     }
1: 
1:     private void lookForReadmeFile(String path) throws IOException {
1:         File readmeFile = new File(path,
1:             DB_README_FILE_NAME);
0:         PrivilegedFileOpsForTests.exists(readmeFile);
0:         PrivilegedFileOpsForTests.isFileEmpty(readmeFile);
1:     }
1: }
1:  
============================================================================
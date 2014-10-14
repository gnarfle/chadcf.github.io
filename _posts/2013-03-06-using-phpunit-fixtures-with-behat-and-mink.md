---
layout: post
title: Using PHPUnit Fixtures with Behat and Mink
created: 1362634700
---
This seems like something that should be fairly trivial but the fixture functionality for PHPUnit's DBUnit is mostly encapsulated in one class which is designed to be used in Test Cases. Since PHP doesn't support multiple inheritance this poses somewhat of a problem if you are using Behat and Mink for BDD but want to be able to easily reset your database after each test. There are some other options out there, using Doctrine or ezComponents, but for me it seemed like DBUnit was the easiest thing to integrate especially since I was already using PHPUnit. So, not really seeing any better solutions, I simply pulled out the items from PHPUnit's Database Test Case class necessary to implement fixtures in a FeatureContext.

To use this class, simply stick it in your features/bootstrap directory, include it in your FeatureContext and make your FeatureContext extend it. If you're not using Mink, you can change this class to extend BehatContext instead. You will want to update the getConnection() method with your test database details and the getDataSet method with your fixture details (see the <a href="http://www.phpunit.de/manual/3.6/en/database.html">PHPUnit manual</a>). In my case I simply prepped a database and dumped each table I wanted to reset to a csv file in features/bootstrap/fixtures. 

Using this class, the defined tables for your dataset will be automatically truncated after each scenario and the data from the csv files reloaded. There you go, a clean database on each test scenario.

<?php
use Behat\MinkExtension\Context\MinkContext;

require_once "PHPUnit/Extensions/Database/TestCase.php";

class BaseFeatureContext extends MinkContext
{
    private $databaseTester;

    public function getConnection()
    {
        $database = 'imebase_test';
        $pdo = new PDO('mysql:dbname=mytestdatabase;host=localhost', 'user', 'password');
        return $this->createDefaultDBConnection($pdo, $database);
    }


    protected function getDataSet()
    {
        $fixtures = array(
            'documents'                 => 'T2_Document',
            'entity'                    => 'T2_Entity',
            'entity_baked'              => 'T2_Entity_Baked',
            'entity_baked_versions'     => 'T2_Entity_Baked_Versions',
            'entity_locations'          => 'T2_Entity_Locations',
            'exam'                      => 'T2_Exam',
            'exam_versions'             => 'T2_Exam_Versions',
            'service'                   => 'T2_Service',
            'service_customfielddata'   => 'T2_Service_CustomFieldData',
            'service_issues'            => 'T2_Service_Issues',
            'service_versions'          => 'T2_Service_Versions'
        );

        $dataSet = new PHPUnit_Extensions_Database_DataSet_CsvDataSet();
        foreach( $fixtures as $file => $table )
        {
            $dataSet->addTable($table, dirname(__FILE__)."/fixtures/{$file}.csv");
        }
        return $dataSet;
    }

    /** @BeforeScenario */
    public function before($event)
    {
        $this->databaseTester = NULL;

        $this->getDatabaseTester()->setSetUpOperation($this->getSetUpOperation());
        $this->getDatabaseTester()->setDataSet($this->getDataSet());
        $this->getDatabaseTester()->onSetUp();
    }

    /** @AfterScenario */
    public function after($event)
    {
        $this->getDatabaseTester()->setTearDownOperation($this->getTearDownOperation());
        $this->getDatabaseTester()->setDataSet($this->getDataSet());
        $this->getDatabaseTester()->onTearDown();

        /**
         * Destroy the tester after the test is run to keep DB connections
         * from piling up.
         */
        $this->databaseTester = NULL;
    }

    protected function getDatabaseTester()
    {
        if (empty($this->databaseTester)) {
            $this->databaseTester = $this->newDatabaseTester();
        }

        return $this->databaseTester;
    }

    protected function newDatabaseTester()
    {
        return new PHPUnit_Extensions_Database_DefaultTester($this->getConnection());
    }

    protected function getSetUpOperation()
    {
        return PHPUnit_Extensions_Database_Operation_Factory::CLEAN_INSERT();
    }

    protected function getTearDownOperation()
    {
        return PHPUnit_Extensions_Database_Operation_Factory::NONE();
    }

    protected function createDefaultDBConnection(PDO $connection, $schema = '')
    {
        return new PHPUnit_Extensions_Database_DB_DefaultDatabaseConnection($connection, $schema);
    }
}
?>

and don't forget to update your FeatureContext as so:

<?php
require_once 'BaseFeatureContext.php';

class FeatureContext extends BaseFeatureContext
{
    ...
}
?>

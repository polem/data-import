Ddeboer Data Import library
===========================
[![Build Status](https://travis-ci.org/ddeboer/data-import.png?branch=master)](https://travis-ci.org/ddeboer/data-import) 
[![Scrutinizer Quality Score](https://scrutinizer-ci.com/g/ddeboer/data-import/badges/quality-score.png?s=41129c80140adc6931288c9df15fb87ec6ea6f8a)](https://scrutinizer-ci.com/g/ddeboer/data-import/) 
[![Code Coverage](https://scrutinizer-ci.com/g/ddeboer/data-import/badges/coverage.png?s=724267091a6d02f83b6c435a431e71d467b361f8)](https://scrutinizer-ci.com/g/ddeboer/data-import/) 
[![Latest Stable Version](https://poser.pugx.org/ddeboer/data-import/v/stable.png)](https://packagist.org/packages/ddeboer/data-import)

Introduction
------------
This PHP library offers a way to read data from, and write data to, a range of
file formats and media. Additionally, it includes tools to manipulate your data.

Features
--------

* Read from and write to CSV files, Excel files, databases, and more.
* Convert between charsets, dates, strings and objects on the fly.
* Build reusable and extensible import workflows.
* Decoupled components that you can use on their own, such as a CSV and Excel
  reader and writer.
* Well-tested code.

Documentation
-------------
* [Installation](#installation)
* [Usage](#usage)
  * [The workflow](#the-workflow)
  * [Readers](#readers)
    - [ArrayReader](#arrayreader)
    - [CsvReader](#csvreader)
    - [DbalReader](#dbalreader)
    - [DoctrineReader](#doctrinereader)
    - [ExcelReader](#excelreader)
    - [Create a reader](#create-a-reader)
  * [Writers](#writers)
    - [ArrayWriter](#arraywriter)
    - [CsvWriter](#csvwriter)
    - [DoctrineWriter](#doctrinewriter)
    - [ExcelWriter](#excelwriter)
    - [ConsoleProgressWriter](#consoleprogresswriter)
    - [CallbackWriter](#callbackwriter)
    - [Create a writer](#create-a-writer)
  * [Filters](#filters)
    - [CallbackFilter](#callbackfilter)
    - [OffsetFilter](#offsetfilter)
    - [ValidatorFilter](#offsetfilter)
  * [Converters](#converters)
    - [Item converters](#item-converters)
      - [MappingItemConverter](#mappingitemconverter)
      - [Create an item converter](#create-an-item-converter)
      - [CallbackItemConverter](#callbackitemconverter)
    - [Value converters](#value-converters)
      - [DateTimeValueConverter](#datetimevalueconverter)
      - [ObjectConverter](#objectconverter)
      - [StringToObjectValueConverter](#stringtoobjectvalueconverter)
      - [ArrayValueConverterMap](#arrayvalueconvertermap)
      - [CallbackValueConverter](#callbackvalueconverter)
  * [Examples](#examples)
    - [Import CSV file and write to database](#import-csv-file-and-write-to-database)
    - [Export to CSV file](#export-to-csv-file)
* [Running the tests](#running-the-tests)
* [License](#license) 

Installation
------------

This library is available on [Packagist](http://packagist.org/packages/ddeboer/data-import).
The recommended way to install it is through [Composer](http://getcomposer.org):

```bash
$ composer require ddeboer/data-import:@stable
```

Then include Composer’s autoloader:

```php
require_once 'vendor/autoload.php';
```

For integration with Symfony2 projects, the [DdeboerDataImportBundle](https://github.com/ddeboer/DdeboerDataImportBundle)
is available.

Usage
-----

Broadly speaking, you can use this library in two ways:

* organize your import process around a [workflow](#workflow), or
* use one or more of the components on their own, such as [readers](#readers),
  [writers](#writers) or [converters](#converters).

### The workflow

Each data import revolves around the workflow and takes place along the following lines:

1. Construct a [reader](#readers).
2. Construct a workflow and pass the reader to it. Add at least one
   [writer](#writers) to the workflow.
3. Optionally, add [filters](#filters), item converters and
   [value converters](#value-converters) to the workflow.
4. Process the workflow. This will read the data from the reader, filter and
   convert the data, and write the output to each of the writers.

In other words, the workflow acts as a [mediator](#http://en.wikipedia.org/wiki/Mediator_pattern)
between a reader and one or more writers, filters and converters.

Schematically:

```php
use Ddeboer\DataImport\Workflow;
use Ddeboer\DataImport\Reader;
use Ddeboer\DataImport\Writer;
use Ddeboer\DataImport\Filter;

$reader = new Reader\...;
$workflow = new Workflow($reader);
$workflow
    ->addWriter(new Writer\...())
    ->addWriter(new Writer\...())
    ->addFilter(new Filter\CallbackFilter(...))
    ->process()
;
```

### Readers

Readers read data that will be imported by iterating over it. This library
includes a handful of readers. Additionally, you can easily
[implement your own](#create-a-reader).

You can use readers on their own, or construct a workflow from them:

```php
$workflow = new Workflow($reader);
```
#### ArrayReader

Reads arrays. Most useful for testing your workflow.

#### CsvReader

Reads CSV files, and is optimized to use as little memory as possible.

```php
use Ddeboer\DataImport\Reader\CsvReader;

$file = new \SplFileObject('/path/to/csv_file.csv');
$reader = new CsvReader($file);

foreach ($reader as $row) {
    // $row will be an array containing the comma-separated elements of the line:
    // array(
    //   0 => 'James',
    //   1 => 'Bond'
    //   etc...
    // )
}
```

##### Column headers

If one of your rows contains column headers, you can read them to make the rows
associative arrays:

```php
$reader->setHeaderRowNumber(0);

foreach ($reader as $row) {
    // $row will now be an associative array:
    // array(
    //   'firstName' => 'James',
    //   'lastName'  => 'Bond'
    //   etc...
    // )
}
```

##### Strict mode

The CSV reader operates in strict mode by default. If the reader encounters a
row where the number of values differs from the number of column headers, an
error is logged and the row is skipped. Retrieve the errors with `getErrors()`.

To disable strict mode, set `$reader->setStrict(false)` after you instantiate
the reader.

Disabling strict mode means:

1. Any rows that contain fewer values than the column headers are simply
   padded with null values.
2. Any additional values in a row that contain more values than the
   column headers are ignored.

Examples where this is useful:

- **Outlook 2010:** which omits trailing blank values
- **Google Contacts:** which exports more values than there are column headers

##### Duplicate headers

Sometimes a CSV file contains duplicate column headers, for instance:

id  | details  | details
--- | -------- | --------
1   | bla      | more bla

By default, a `DuplicateHeadersException` will be thrown if you call
`setHeaderRowNumber(0)` on this file. You can handle duplicate columns in
one of three ways:
* call `setColumnHeaders(['id', 'details', 'details_2'])` to specify your own
  headers
* call `setHeaderRowNumber` with the `CsvReader::DUPLICATE_HEADERS_INCREMENT`
  flag to generate incremented headers; in this case: `id`, `details` and
  `details1`
* call `setHeaderRowNumber` with the `CsvReader::DUPLICATE_HEADERS_MERGE` flag
  to merge duplicate values into arrays; in this case, the first row’s values
  will become: `[ 'id' => 1, 'details' => [ 'bla', 'more bla' ] ]`.

#### DbalReader

Reads data through [Doctrine’s DBAL](http://www.doctrine-project.org/projects/dbal.html).
Your project should include Doctrine’s DBAL package:

```bash
$ composer require doctrine/dbal
```

```php
use Ddeboer\DataImport\Reader\DbalReader;

$reader = new DbalReader(
    $connection, // Instance of \Doctrine\DBAL\Connection
    'SELECT u.id, u.username, g.name FROM `user` u INNER JOIN groups g ON u.group_id = g.id'
);
```

#### DoctrineReader

Reads data through the [Doctrine ORM](http://www.doctrine-project.org/projects/orm.html):

```php
use Ddeboer\DataImport\Reader\DoctrineReader;

$reader = new DoctrineReader($entityManager, 'Your\Namespace\Entity\User');
```

#### ExcelReader

Acts as an adapter for the [PHPExcel library](http://phpexcel.codeplex.com/). Make sure
to incude that library in your project:

```bash
$ composer require phpoffice/phpexcel
```

Then use the reader to open an Excel file:

```php
use Ddeboer\DataImport\Reader\ExcelReader;

$file = new \SplFileObject('path/to/ecxel_file.xls');
$reader = new ExcelReader($file);
```

#### Create a reader

You can create your own data reader by implementing the
[ReaderInterface](/src/Ddeboer/DataImport/Reader/ReaderInterface.php).

### Writers

#### ArrayWriter

Resembles the [ArrayReader](#arrayreader). Probably most useful for testing
your workflow.

#### CsvWriter

Writes CSV files:

```php
use Ddeboer\DataImport\Writer\CsvWriter;

$file = new \SplFileObject('output.csv', 'w');
$writer = new CsvWriter($file);

// Write column headers:
$writer->writeItem(array('first', 'last'));

$writer
    ->writeItem(array('James', 'Bond'))
    ->writeItem(array('Auric', 'Goldfinger'))
    ->finish();
```

#### DoctrineWriter

Writes data through Doctrine:

```php
use Ddeboer\DataImport\Writer\DoctrineWriter;

$writer = new DoctrineWriter($entityManager, 'YourNamespace:Employee');
$writer
    ->prepare()
    ->writeItem(
        array(
            'first' => 'James',
            'last'  => 'Bond'
        )
    )
    ->finish();
```

#### ExcelWriter

Writes data to an Excel file. It requires the PHPExcel package:

```bash
$ composer require phpoffice/phpexcel
```

```php
use Ddeboer\DataImport\Writer\ExcelWriter;

$file = new \SplFileObject('data.xlsx', 'w');
$writer = new ExcelWriter($file);

$writer
    ->prepare()
    ->writeItem(array('first', 'last'))
    ->writeItem(array('first' => 'James', 'last' => 'Bond'))
    ->finish();
```

You can specify the name of the sheet to write to:

```php
$writer = new ExcelWriter($file, 'My sheet');
```

You can open an already existing file and add a sheet to it:

```php
$file = new \SplFileObject('data.xlsx', 'a');   // Open file with append mode
$writer = new ExcelWriter($file, 'New sheet');
```

If you wish to overwrite an existing sheet instead, specify the name of the
existing sheet:

```php
$writer = new ExcelWriter($file, 'Old sheet');
```

#### ConsoleProgressWriter

This writer displays import progress when you start the workflow from the 
command-line. It requires Symfony’s Console component:

```bash
$ composer require symfony/console
```

```php
use Ddeboer\DataImport\Writer\ConsoleProgressWriter;
use Symfony\Component\Console\Output\ConsoleOutput;

$output = new ConsoleOutput(...);
$progressWriter = new ConsoleProgressWriter($output, $reader);

// Most useful when added to a workflow
$workflow->addWriter($progressWriter);
```

#### CallbackWriter

Instead of implementing your own writer, you can use the quick solution the
CallbackWriter offers:

```php
use Ddeboer\DataImport\Writer\CallbackWriter;

$workflow->addWriter(new CallbackWriter(function ($row) use ($storage) {
    $storage->store($row);
}));
```

#### Create a writer

Build your own writer by implementing the
[WriterInterface](/src/Ddeboer/DataImport/Writer/WriterInterface.php).

### Filters

A filter decides whether data input is accepted into the import process. 

#### CallbackFilter

The CallbackFilter wraps your callback function that determines whether
data should be accepted. The data input is accepted only if the function 
returns `true`.

```php
use Ddeboer\DataImport\Filter\CallbackFilter;

// Don’t import The Beatles
$filter = new CallbackFilter(function ($data) {
    return ('The Beatles' != $data['name']);
});

$workflow->addFilter($filter);
```

#### OffsetFilter

OffsetFilter allows you to

* skip a certain amount of items from the beginning
* process only specified amount of items (and skip the rest)

You can combine these two parameters to process a slice from the middle of the
data, like rows 5-7 of a CSV file with ten rows.

OffsetFilter is configured by its constructor:
`new OffsetFilter($offset = 0, $limit = null)`. Note: `$offset` is a 0-based index.

```php
use Ddeboer\DataImport\Filter\OffsetFilter;

// Default implementation is to start from the beginning without maximum count
$filter = new OffsetFilter(0, null);
$filter = new OffsetFilter(); // You can omit both parameters

// Start from the third item, process to the end
$filter = new OffsetFilter(2, null);
$filter = new OffsetFilter(2); // You can omit the second parameter

// Start from the first item, process max three items
$filter = new OffsetFilter(0, 3);

// Start from the third item, process max five items (items 3 - 7)
$filter = new OffsetFilter(2, 5);
```

#### ValidatorFilter

It’s a common use case to validate the data before you save it to the database.
This is exactly what the ValidatorFilter does. To use it, include Symfony’s
Validator component in your project:

```bash
$ compose require symfony/validator
```

The ValidatorFilter works as follows:

```php
use Ddeboer\DataImport\Filter\ValidatorFilter;

$filter = new ValidatorFilter($validator);
$filter->add('email', new Assert\Email());
$filter->add('sku', new Assert\NotBlank());
```

The default behaviour for the validator is to collect all violations and skip
each invalid row. If you want to stop on the first failing row you can call
`ValidatorFilter::throwExceptions()`, which throws a
[ValidationException](/src/Ddeboer/DataImport/Exception/ValidationException.php)
containing the line number and the violation list.

### Item converters

#### MappingItemConverter

Use the MappingItemConverter to add mappings to your workflow. Your keys from
the input data will be renamed according to these mappings. Say you have input data:

```php
$data = array(
    array(
        'foo' => 'bar',
        'baz' => array(
            'some' => 'value'
        )
    )
);
```

You can map the keys `foo` and `baz` in the following way:

```php

use Ddeboer\DataImport\ItemConverter\MappingItemConverter;

$converter = new MappingItemConverter();
$converter
    ->addMapping('foo', 'fooloo')
    ->addMapping('baz', array('some' => 'else'));

$workflow->addItemConverter($converter)
    ->process();
```

Your output data will now be:

```php
array(
    array(
        'fooloo' => 'bar',
        'baz'    => array(
            'else' => 'value'
        )
    )
);
```

#### Create an item converter

Implement `ItemConverterInterface` to create your own item converter:

```php
use Ddeboer\DataImport\ItemConverter\ItemConverterInterface;

class MyItemConverter implements ItemConverterInterface 
{
    public function convert($item)
    {
        // Do your conversion and return updated $item
        return $changedItem;
    }
}
```

#### CallbackItemConverter

Instead of implementing your own item converter, you can use a callback:

```php
use Ddeboer\DataImport\ItemConverter\CallbackItemConverter;

// Use a fictional $translator service to translate each value
$converter = new CallbackItemConverter(function ($item) use ($translator) {
    foreach ($item as $key => $value) {
        $item[$key] = $translator->translate($value);
    }
  
    return $row;
});
```

### Value converters

Value converters are used to convert specific fields (e.g., columns in database).

#### DateTimeValueConverter

Converts a date representation in a format you specify into a `DateTime` object:

```php
use Ddeboer\DataImport\ValueConverter\DateTimeValueConverter;

$converter = new DateTimeValueConverter('d/m/Y H:i:s');
$workflow->addValueConverter('my_date_field', $converter);
```

#### ObjectConverter

Converts an object into a scalar value. To use this converter, you must include
Symfony’s PropertyAccess component in your project:

```bash
$ composer require symfony/property-access
```

##### Using __toString()

If your object has a `__toString()` method, that value will be used:

```php
use Ddeboer\DataImport\ValueConverter\ObjectConverter;

class SecretAgent
{
    public function __toString()
    {
        return '007';
    }
}

$converter = new ObjectConverter();
$string = $converter->convert(new SecretAgent());   // $string will be '007'
```

##### Using object accessors

If your object has no `__toString()` method, its accessors will be called
instead:

```php
class Villain
{
    public function getName()
    {
        return 'Bad Guy';
    }
}

class Organization
{
    public funtion getVillain()
    {
        return new Villain();
    }
}

use Ddeboer\DataImport\ValueConverter\ObjectConverter;

$converter = new ObjectConverter('villain.name');
$string = $converter->convert(new Organization());   // $string will be 'Bad Guy'
```

#### StringToObjectConverter

Looks up an object in the database based on a string value:

```php
use Ddeboer\DataImport\ValueConverter\StringToObjectConverter;

$converter = new StringToObjectConverter($repository, 'name');
$workflow->addValueConverter('input_name', $converter);
```

#### CallbackValueConverter

Use this if you want to save the trouble of writing a dedicating class:

```php
use Ddeboer\DataImport\ValueConverter\CallbackValueConverter;

$callable = function ($item) {
    return implode(',', $item);
};

$converter = new CallbackValueConverter($callable);
$output = $converter->convert(array('foo', 'bar')); // $output will be "foo,bar"
```

### Examples

#### Import CSV file and write to database

This example shows how you can read data from a CSV file and write that to the
database.

Assume we have the following CSV file:

```csv
event;beginDate;endDate
Christmas;20131225;20131226
New Year;20131231;20140101
```

And we want to write this data to a Doctrine entity:

```php
namespace MyApp;

use Doctrine\ORM\Mapping as ORM;

/**
 * @ORM\Entity
 */
class Event
{
    /**
     * @ORM\Column()
     */
    protected $event;

    /**
     * @ORM\Column(type="datetime")
     */
    protected $beginDate;

    /**
     * @ORM\Column(type="datetime")
     */
    protected $endDate;

    public function setEvent($event)
    {
        $this->event = $event;
    }

    public function setBeginDate($date)
    {
        $this->beginDate = $date;
    }

    public function setEndDate($date)
    {
        $this->endDate = $date;
    }

    // And some getters
}
```

Then you can import the CSV and save it as your entity in the following way.

```php
use Ddeboer\DataImport\Workflow;
use Ddeboer\DataImport\Reader\CsvReader;
use Ddeboer\DataImport\Writer\DoctrineWriter;
use Ddeboer\DataImport\ValueConverter\DateTimeValueConverter;

// Create and configure the reader
$file = new \SplFileObject('input.csv');
$csvReader = new CsvReader($file);

// Tell the reader that the first row in the CSV file contains column headers
$csvReader->setHeaderRowNumber(0);

// Create the workflow from the reader
$workflow = new Workflow($csvReader);

// Create a writer: you need Doctrine’s EntityManager.
$doctrineWriter = new DoctrineWriter($entityManager, 'MyApp:Event');
$workflow->addWriter($doctrineWriter);

// Add a converter to the workflow that will convert `beginDate` and `endDate`
// to \DateTime objects
$dateTimeConverter = new DateTimeValueConverter('Ymd');
$workflow
    ->addValueConverter('beginDate', $dateTimeConverter)
    ->addValueConverter('endDate', $dateTimeConverter);

// Process the workflow
$workflow->process();
```

#### Export to CSV file

This example shows how you can export data to a CSV file.

```php
use Ddeboer\DataImport\Workflow;
use Ddeboer\DataImport\Reader\ArrayReader;
use Ddeboer\DataImport\Writer\CsvWriter;
use Ddeboer\DataImport\ValueConverter\CallbackValueConverter;

// Your input data
$reader = new ArrayReader(array(
    array(
        'first',        // This is for the CSV header
        'last',
    array(
        'first' => 'james',
        'last'  => 'Bond'
    ),
    array(
        'first' => 'hugo',
        'last'  => 'Drax'
    )
);

// Create the workflow from the reader
$workflow = new Workflow($reader);

// Add the writer to the workflow
$file = new \SplFileObject('output.csv', 'w');
$writer = new Writer($file);
$workflow->addWriter($writer);

// As you can see, the first names are not capitalized correctly. Let's fix
// that with a value converter:
$converter = new CallbackValueConverter(function ($input) {
    return ucfirst($input);
});
$workflow->addValueConverter('first', $converter);

// Process the workflow
$workflow->process();
```

This will write a CSV file `output.csv` where the first names are capitalized:

```csv
first;last
James;Bond
Hugo;Drax
```

ArrayValueConverterMap
----------------------

The ArrayValueConverterMap is used to filter values of a multi-level array.

The converters defined in the list are applied on every data-item's value that match the defined array_keys.

```php
//...
$data = array(
    'products' => array(
        0 => array(
            'name' => 'some name',
            'price' => '€12,16',
        ),
        1 => array(
            'name' => 'some name',
            'price' => '€12,16',
        )
    )
);

// ...
// create the workflow and reader etc.
// ...

$workflow->addValueConverter(new ArrayValueConverterMap(array(
    'name' => array(new CharsetValueConverter('UTF-8', 'UTF-16')), // encode to UTF-8
    'price' => array(new CallbackValueConverter(function ($input) {
        return str_replace('€', '', $intput); // remove € char
    }),
)));

// ..
// after filtering data looks as follows
$data = array(
    'products' => array(
        0 => array(
            'name' => 'some name', // in UTF-8
            'price' => '12,16',
        ),
        1 => array(
            'name' => 'some name',
            'price' => '12,16',
        )
    )
);
```

Running the tests
-----------------

Clone this repository:

```bash
$ git clone https://github.com/ddeboer/data-import.git
$ cd data-import
```

Install dev dependencies:

```bash
$ composer install --dev
```

And run PHPUnit:

```bash
$ phpunit
```

License
-------

DataImport is released under the MIT license. See the [LICENSE](LICENSE) file for details.

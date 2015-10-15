Behat-TableAssert simplifies working with and asserting against all sorts of tabular data in Behat. It currently
supports Behat v2 and v3. It has a few extra features when used with Mink, but works happily without.

[![License](https://poser.pugx.org/ingenerator/behat-tableassert/license.svg)](https://packagist.org/packages/ingenerator/behat-tableassert)
[![Build Status](https://travis-ci.org/ingenerator/behat-tableassert.svg?branch=master)](https://travis-ci.org/ingenerator/behat-tableassert)
[![Latest Stable Version](https://poser.pugx.org/ingenerator/behat-tableassert/v/stable.svg)](https://packagist.org/packages/ingenerator/behat-tableassert)
[![Latest Unstable Version](https://poser.pugx.org/ingenerator/behat-tableassert/v/unstable.svg)](https://packagist.org/packages/ingenerator/behat-tableassert)
[![HHVM Status](http://hhvm.h4cc.de/badge/ingenerator/behat-tableassert.svg?branch=master)](http://hhvm.h4cc.de/package/ingenerator/behat-tableassert)
[![Scrutinizer Quality Score](https://scrutinizer-ci.com/g/ingenerator/behat-tableassert/badges/quality-score.png?s=ad84e95fc2405712f88a96d89b4f31dfe5c80fae)](https://scrutinizer-ci.com/g/ingenerator/behat-tableassert/)
[![Total Downloads](https://poser.pugx.org/ingenerator/behat-tableassert/downloads.svg)](https://packagist.org/packages/ingenerator/behat-tableassert)


# Installing Behat-TableAssert

`$> composer require --dev ingenerator/behat-tableassert`

This package isn't a behat extension as such, so you don't need to configure anything further - the classes will be
picked up by the composer autoloader.

# Parsing tables from external data

Often you need to compare tabular data from your feature file with a table generated by your application. For example,
your application might produce an HTML table on screen or a CSV file for download.

You can use one of the provided parsers to load data from a variety of sources into a standard Behat TableNode.

### Parsing HTML tables

The HTML table parser expects a well-formed table with a `<thead>` containing a single row of column headings and zero
or more rows in the `<tbody>`. All rows should have the same number of columns, which can be any mix of `<th>` and
`<td>` elements.

> If rows contain cells with a colspan attribute, the parsed table will contain the cell content in the first column
  and the remaining colspan width will be padded with cells containing `...`.

> If rows have unexpected extra or missing cells (eg due to rendering errors) the parsed table will have extra cells
  added at the end of each row to make all rows the same width. We're not smart enough to pad in the middle of a row,
  so columns to the right of a missing cell will not line up with the columns above them.

> If you want to ignore some rows in the table - for example if they provide additional human-readable
  headings or other irrelevant presentational content - you can mark them up like this `<tr data-behat-table="ignore">`
  and the parser will act as though they don't exist.


```php
use \Ingenerator\BehatTableAssert\TableParser\HTMLTable;
// Without mink you can parse from a string containing just the <table> tag and children
$html  = '<table><thead><tr><th>Col</th></tr></thead><tbody><tr><td>Cell</td></tr></tbody></table>';
$table = HTMLTable::fromHTMLString($html);

// With mink you can parse from a NodeElement - again this must be a <table> element
$assert = $this->getMink()->assertSession();
$table = HTMLTable::fromMinkTable($assert->elementExists('css', 'table'));
```

### Parsing CSV tables

The CSV table parser expects standard CSV content. As with HTML, the parsed table will be padded at the right hand side
to ensure all rows have the same number of columns but this may result in columns not lining up if there are missing
columns towards the left of a row.

```php
use \Ingenerator\BehatTableAssert\TableParser\CSVTable;
// Without mink you can parse from a stream
$file = fopen('/some/csv/file', 'r');
$table = CSVTable::::fromStream($file);
fclose($file);

// Or a string if that's easier
$table = CSVTable::::fromString(file_get_contents('/some/csv/file');

// With mink you can also parse directly from a server response *if*:
//   - the CSV is rendered to the browser as the full raw content of the response
//   - the response has a text/csv content-type header
//   - either you omit any `Content-Disposition: Attachment` header, or you're
//     using a goutte-like driver that ignores it
//
// If your driver respects a Content-Disposition attachment header then it will save
// your server response to disk somewhere just like a normal browser - you will need
// to locate the downloaded file and use the string or stream parsing option.
$table = CSVTable::fromMinkResponse($this->getMink()->getSession());
```


# Comparing tables

Once you've parsed your table (or constructed it from an array of your choice) you will want to compare the values
to whatever you've provided in your feature specification.

### Simple comparisons

```php
$tableassert = new \Ingenerator\BehatTableAssert\AssertTable;

// Throws unless tables are identical - same columns, in same order, containing same data
$tableassert->isSame($expected, $actual, 'Optional extra message');

// Throws unless tables are equal - same columns containing same data, but in any order
$tableassert->isEqual($expected, $actual, 'Optional extra message');

// Throws unless actual table contains at least all the specified columns with the same data,
// ignoring any extra columns.
// Useful if your application produces a large dataset but you only want to specify a few
// values in your feature file.
$tableassert->containsColumns($expected, $actual, 'Optional extra message');
```

### Custom comparison

The simple comparisons all rely on a strict string comparison between the cell values in each table. Sometimes that's
not flexible enough and you need to implement custom logic to decide whether two values are equivalent. We have your
back. The `isComparable` assertion takes a full array of options for the diff engine, including an array of
`comparators` -  custom callables that return TRUE if the expected and actual value are the same, or FALSE if not.

One common case would be where your application generates reports with fixed dates but you want to use relative dates
in the scenario, or where you don't care if there are small rounding errors in the reported data.

```gherkin
  Then I should see a report containing:
    | date      | status | uptime |
    | today     | good   | 99.9   |
    | yesterday | poor   | 50.1   |
    | -2 days   | fair   | 99.7   |
```

```php
  /**
   * @Then /^I should see a report containing$/
   */
  function assertReport(TableNode $expected)
  {
    $actual = \Ingenerator\BehatTableAssert\TableParser\CSVTable::fromMinkResponse($this->getMink()->getSession());
    $assert = new \Ingenerator\BehatTableAssert\AssertTable;
    $assert->isComparable(
      $expected,
      $actual,
      [
        'comparators' => [
          'date'   => function ($expected, $actual) { return new \DateTime($expected) == new \DateTime($actual); },
          'uptime' => function ($expected, $actual) { return round($expected, 0) == round($actual, 0); },
        ],
        'ignoreColumnSequence' => TRUE,
        'ignoreExtraColumns' => TRUE
      ]
    );
  }
```

### Assertion failures

Failed assertions will throw a `\Ingenerator\BehatTableAssert\TableAssertionFailureException`. If there are structural
differences between the tables - for example, missing columns, columns out of sequence, or unexpected extra columns, you
will see something like this:

```
Failed asserting that two tables were equal: some custom message

Structural Differences:
-----------------------
 - Missing columns: 'date' (got 'day', 'status', 'uptime')
 - Unexpected columns: 'day'
```

If the structure is OK but there are differences in individual cell values you will see a message with the actual table
values received and invalid values highlighted inline. Columns and rows containing errors are marked with an X, and
every invalid row is followed by an additional row showing the expected value only for cells that failed, and an ~ where
the cell value was as-expected.

```
Failed asserting that two tables were equal: some other custom message

|   | X date       | X status | uptime |
|   |   2015-10-09 |   good   | 99.8   |
| X | X 2015-09-01 |   poor   | 50.1   |
| i | ^ 2015-10-08 |   ~      | ~      |
| X |   2015-10-07 | X good   | 99.7   |
| i |   ~          | ^ fair   | ~      |
```

# Contributing

Contributions are welcome - please see [CONTRIBUTING.md](CONTRIBUTING.md) before starting work on any contribution.

# Contributors

This package has been sponsored by [inGenerator Ltd](http://www.ingenerator.com)

* Andrew Coulton [acoulton](https://github.com/acoulton) - Lead developer

# Licence

Licensed under the [BSD-3-Clause Licence](LICENSE)

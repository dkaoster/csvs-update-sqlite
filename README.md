# csvs-update-sqlite

Convert CSV files into a SQLite database. Browse and publish that SQLite database with [Datasette](https://github.com/simonw/datasette).

Based on [csvs-to-sqlite](https://github.com/simonw/csvs-to-sqlite).

Basic usage:

    csvs-update-sqlite myfile.csv mydatabase.db

This will create a new SQLite database called `mydatabase.db` containing a
single table, `myfile`, containing the CSV content.

You can provide multiple CSV files:

    csvs-update-sqlite one.csv two.csv bundle.db

The `bundle.db` database will contain two tables, `one` and `two`.

This means you can use wildcards:

    csvs-update-sqlite ~/Downloads/*.csv my-downloads.db

If you pass a path to one or more directories, the script will recursively
search those directories for CSV files and create tables for each one.

    csvs-update-sqlite ~/path/to/directory all-my-csvs.db

## Handling TSV (tab-separated values)

You can use the `-s` option to specify a different delimiter. If you want
to use a tab character you'll need to apply shell escaping like so:

    csvs-update-sqlite my-file.tsv my-file.db -s $'\t'

## Refactoring columns into separate lookup tables

Let's say you have a CSV file that looks like this:

    county,precinct,office,district,party,candidate,votes
    Clark,1,President,,REP,John R. Kasich,5
    Clark,2,President,,REP,John R. Kasich,0
    Clark,3,President,,REP,John R. Kasich,7

([Real example taken from the Open Elections project](https://github.com/openelections/openelections-data-sd/blob/master/2016/20160607__sd__primary__clark__precinct.csv))

You can now convert selected columns into separate lookup tables using the new
`--extract-column` option (shortname: `-c`) - for example:

    csvs-update-sqlite openelections-data-*/*.csv \
        -c county:County:name \
        -c precinct:Precinct:name \
        -c office -c district -c party -c candidate \
        openelections.db

The format is as follows:

    column_name:optional_table_name:optional_table_value_column_name

If you just specify the column name e.g. `-c office`, the following table will
be created:

    CREATE TABLE "office" (
        "id" INTEGER PRIMARY KEY,
        "value" TEXT
    );

If you specify all three options, e.g. `-c precinct:Precinct:name` the table
will look like this:

    CREATE TABLE "Precinct" (
        "id" INTEGER PRIMARY KEY,
        "name" TEXT
    );

The original tables will be created like this:

    CREATE TABLE "ca__primary__san_francisco__precinct" (
        "county" INTEGER,
        "precinct" INTEGER,
        "office" INTEGER,
        "district" INTEGER,
        "party" INTEGER,
        "candidate" INTEGER,
        "votes" INTEGER,
        FOREIGN KEY (county) REFERENCES County(id),
        FOREIGN KEY (party) REFERENCES party(id),
        FOREIGN KEY (precinct) REFERENCES Precinct(id),
        FOREIGN KEY (office) REFERENCES office(id),
        FOREIGN KEY (candidate) REFERENCES candidate(id)
    );

They will be populated with IDs that reference the new derived tables.

## Installation

    $ pip install csvs-update-sqlite

## csvs-update-sqlite --help

<!-- [[[cog
import cog
from csvs_update_sqlite import cli
from click.testing import CliRunner
runner = CliRunner()
result = runner.invoke(cli.cli, ["--help"])
help = result.output.replace("Usage: cli", "Usage: csvs-update-sqlite")
cog.out(
    "```\n{}\n```".format(help)
)
]]] -->
```
Usage: csvs-update-sqlite [OPTIONS] PATHS... DBNAME

  PATHS: paths to individual .csv files or to directories containing .csvs

  DBNAME: name of the SQLite database file to create

Options:
  -s, --separator TEXT            Field separator in input .csv
  -q, --quoting INTEGER           Control field quoting behavior per csv.QUOTE_*
                                  constants. Use one of QUOTE_MINIMAL (0),
                                  QUOTE_ALL (1), QUOTE_NONNUMERIC (2) or
                                  QUOTE_NONE (3).

  --skip-errors                   Skip lines with too many fields instead of
                                  stopping the import

  --replace-tables                Replace tables if they already exist
  --update-tables                 Manages an extra table .csvs-meta that keeps
                                  track of each CSV file and the checksum of the
                                  file. On subsequent runs, the CSVs will be
                                  compared against the checksum in the table to
                                  see what has updated, and only those specific
                                  tables will be replaced.

  -t, --table TEXT                Table to use (instead of using CSV filename)
  -c, --extract-column TEXT       One or more columns to 'extract' into a
                                  separate lookup table. If you pass a simple
                                  column name that column will be replaced with
                                  integer foreign key references to a new table
                                  of that name. You can customize the name of
                                  the table like so:     state:States:state_name
                                  
                                  This will pull unique values from the 'state'
                                  column and use them to populate a new 'States'
                                  table, with an id column primary key and a
                                  state_name column containing the strings from
                                  the original column.

  -d, --date TEXT                 One or more columns to parse into ISO
                                  formatted dates

  -dt, --datetime TEXT            One or more columns to parse into ISO
                                  formatted datetimes

  -df, --datetime-format TEXT     One or more custom date format strings to try
                                  when parsing dates/datetimes

  -pk, --primary-key TEXT         One or more columns to use as the primary key
  -f, --fts TEXT                  One or more columns to use to populate a full-
                                  text index

  -i, --index TEXT                Add index on this column (or a compound index
                                  with -i col1,col2)

  --shape TEXT                    Custom shape for the DB table - format is
                                  csvcol:dbcol(TYPE),...

  --filename-column TEXT          Add a column with this name and populate with
                                  CSV file name

  --fixed-column <TEXT TEXT>...   Populate column with a fixed string
  --fixed-column-int <TEXT INTEGER>...
                                  Populate column with a fixed integer
  --fixed-column-float <TEXT FLOAT>...
                                  Populate column with a fixed float
  --no-index-fks                  Skip adding index to foreign key columns
                                  created using --extract-column (default is to
                                  add them)

  --no-fulltext-fks               Skip adding full-text index on values
                                  extracted using --extract-column (default is
                                  to add them)

  --just-strings                  Import all columns as text strings by default
                                  (and, if specified, still obey --shape,
                                  --date/datetime, and --datetime-format)

  --version                       Show the version and exit.
  --help                          Show this message and exit.

```
<!-- [[[end]]] -->

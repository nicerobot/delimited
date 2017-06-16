# A sane format for raw data files

Raw data is incredibly easy to format but can be dauntingly difficult to interpret everyone's formatting whims. A few simple standards can go a long way. The structure of the raw data isn't enough. Besides, the [CSV spec](https://tools.ietf.org/html/rfc4180), though relatively simple, is still quite often misinterpreted or even entirely ignored.

When processing a raw file, the schema of the file is almost always needed. Having to read documentation, all too often saved as a PDF, is nonsensical when virtually all the information about the contents of the file can easily be encoded into the file itself by the creator of the content.

# How?

- No raw file should ever be created that doesn't contain a header row.
- All raw files should be TSV.
- All rows should be single line.
- Every `\` is encoded as `\\`.
- All non-printable characters are hex-encoded as `\0x__`.
- The entire contents of a field is the value of the field. That is, if there are quotes in the field, the quotes are part of the value. There is no such thing as a quoted value.

# Header flags

Each column header should include hints as to the nature of the column.

- `*` primary key
- `=[X]` unique key
- `!` required (not null)
- `[tdifs]` data type t=text d=date i=integer f=float s=timestamp
- `([!]N[,M])` fixed or maximum length/precision
- `{text}` value that represents null

## date/time:

- `[d]` is always UTC YYYYMMDD.
- `[s]` is always UTC Epoch seconds (possibly with fractional seconds).

# Example

Consider this space-delimited header example:

    pk1(!2)* pk2[i]* unq1a[i]= unq1b[t]= unq1c2a(3)[t]==A unq2b[i]=A unq2c[i]=A colA[t](12){.} colB[f]{-}{.}{} colC[i]! colD[d]

- The primary key fields are ordered left to right so as not to require an order specifier.
- The uniqueness flag's identifier is the trailing flag and can be any single character for grouping unique keys.
- All uniqueness flags without an identifier are grouped into the same unique key.
- Uniqueness columns are ordered left to right.

How to interpret each header hint:

- `pk1(!2)*` - The first primary key field. Is text, by default, and must always be two characters.
- `pk2[i]*` - The second primary key field. An integer.
- `unq1a[i]=` - The first field of the "default" unique key. An integer.
- `unq1b[t]=` - The second field of the "default" unqique key. Text.
- `unq1c2a(3)[t]==A` - Both the third field of the "default" unique key and the first field of unique key "A". Text with three character maximum length.
- `unq2b[i]=A` - The second field of unique key "A". An integer.
- `unq2c[i]=A` - The third field of unique key "A". An integer.
- `colA[t](12){.}` - A text column with a maximum of twelve characters. `.` represents that the value is null. This implies that an empty field is not null.
- `colB[f]{-}{.}{}` - A float column where `-`, `.`, and the empty field represent null values. Note the significance of this being a floating point field possibly containing values that are not floats which would derail automated data type attempts.
- `colC[i]!` - A required integer field.
- `colD[d]` - A date.

The above is equivalent to (assuming a somewhat generic SQL database):

    create table ... (
      pk1     char(2)      not null,
      pk2     integer      not null,
      unq1a   integer      not null,
      unq1b   text         not null,
      unq1c2a varchar(3)   not null,
      unq2b   integer      not null,
      unq2c   integer      not null,
      colA    varchar(12),
      colB    float,
      colC    integer not null,
      colD    date,
      primary key (pk1, pk2)
    );

    create unique index ... (unq1a,unq1c,unq1c2a);
    create unique index ... (unq1c2a,unq2b,unq2c);

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

Column headings are specified like URL query parameters.

    column_name?parameter...

where parameters can be (not always requiring values)

- `pk` primary key (is requied by default)
- `uk[=name[,order]]` optionally named unique key (required/not-null by default)
- `[type][!][=length|precision|format]` optionally lengthed and required data type
  - `!` after the type (before `=`) means required (not null)
  - `text` or `t` `=length[!]` text is the default type
    - `!` after length means fixed length
  - `date` or `d` `=format` defaults to `Ymd`
  - `integer` or `i` `=length`
  - `float` or `f` `=length[,precision]`
  - `timestamp` or `s` `=format` defaults to `YmdTHMS`
- `null[=re]` the regular expression that represents nulls. defaults to empty string

# Example

Consider this space-delimited header example:

    a?pk&2 b?pk&i c?i&uk d?t&uk e?t=3&uk=A f?i&uk=A,1 g?i&uk=A h?12?null=. i?f&n=[-.]|^$ j?i! k?d

- The primary key fields are ordered left to right so as not to require an order specifier.
- The uniqueness flag's identifier is the trailing flag and can be any single character for grouping unique keys.
- All uniqueness flags without an identifier are grouped into the same unique key.
- Uniqueness columns are ordered left to right.

How to interpret each header hint:

- `a?pk&2` - The first primary key field. Is text, by default, and must always be two characters.
- `b?pk&i` - The second primary key field. An integer.
- `c?i` - The first field of the "default" unique key. An integer.
- `d?t` - The second field of the "default" unqique key. Text.
- `e?t=3&uk=A` - Both the third field of the "default" unique key and the first field of unique key "A". Text with three character maximum length.
- `f?i&uk=A` - The second field of unique key "A". An integer.
- `g?i&uk=A` - The third field of unique key "A". An integer.
- `h?12?null=.` - A text column with a maximum of twelve characters. `.` represents that the value is null. This implies that an empty field is not null.
- `i?f&n=[-.]|^$` - A float column where `-`, `.`, and the empty field represent null values. Note the significance of this being a floating point field possibly containing values that are not floats which would derail automated data type attempts.
- `j?i!` - A required integer field.
- `k?d` - A date.

The above is equivalent to (assuming a somewhat generic SQL database):

    create table ... (
      a  char(2)      not null,
      b  integer      not null,
      c  integer      not null,
      d  text         not null,
      e  varchar(3)   not null,
      f  integer      not null,
      g  integer      not null,
      h  varchar(12),
      i  float,
      j  integer not null,
      k  date,
      primary key (a, b)
    );

    create unique index ... (c,d);
    create unique index A (f,e,g);

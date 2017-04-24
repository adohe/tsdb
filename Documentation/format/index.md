# Chunks Disk Format

The following describes the format of the `index` file found in each block directory.

```
 ┌────────────────────────────┬──────────────────┐
 │ magic <4 byte, 0xBAAAD700> │ version <1 byte> │
 ├────────────────────────────┴──────────────────┤
 │                    Body ...                   │
 └───────────────────────────────────────────────┘
```

## Body (v1)

The body is split into the following parts:

```
 ┌───────────────────────────────────────────────┐
 │                  Symbol Table                 │
 ├───────────────────────────────────────────────┤
 │                     Series                    │
 ├───────────────────────────────────────────────┤
 │                  Label Index                  │
 ├───────────────────────────────────────────────┤
 │                    Postings                   │
 ├───────────────────────────────────────────────┤
 │                    Body ...                   │
 ├───────────────────────────────────────────────┤
 │                    Body ...                   │
 └───────────────────────────────────────────────┘
```


Various pieces of data are often enclosed in a *section*. Sectioned data is always enclosed in the following format, which we will subsequently elide for brevity:

```
 ┌──────────────────┬──────────────┬────────────┬────────────────┐
 │ version <1 byte> │ len <4 byte> │  data ...  │ CRC32 <4 byte> │
 └──────────────────┴──────────────┴────────────┴────────────────┘
```

`version` provides a versioning number for the contents and also implies the existance of length and checksum fields.  
`len` denotes the length of the data section.


### Symbol Table

The symbol table holds all strings encountered in our index. All other index sections just reference strings in the table as they are highly repetitive.

#### v1, section(`1`)

The section contains a sequence of the raw string data, each prefixed with the string's length.
Strings are referenced by pointing to the beginning of their length field. The strings are sorted in lexicographically ascending order.

```
 ┌─────────────────────┬───────────────┐
 │ len(str_1) <varint> │ str_1 <bytes> │
 ├─────────────────────┴───────────────┤
 │                . . .                │
 ├─────────────────────┬───────────────┤
 │ len(str_n) <varint> │ str_1 <bytes> │
 └─────────────────────┴───────────────┘
```


### Series

#### v1, section(`1`)

The section contains a sequence of series that hold the label set of the series as well as the chunks within the block. The series are sorted lexicographically by their label sets.  
The file offset to the beginning of a series serves as the series' ID in all subsequent references. Thereby, a sorted list of series IDs implies a lexicographically sorted list of series label sets.

```
 ┌────────────┬─────────┬────────────┐
 │  series_1  │  . . .  │  series_n  │
 └────────────┴─────────┴────────────┘
```

Every series holds a list of label pairs and chunks. The label pairs reference the symbol table and the chunks an address in one of the block's chunk files.

```
 ┌──────────────────┬────────────────────────────────────────────────────────────────────────┐
 │                  │ ┌────────────────────────┬─────────────────────────┐                   │
 │ #labels <varint> │ │ ref(l_i.name) <varint> │ ref(l_i.value) <varint> │ ...               │
 │                  │ └────────────────────────┴─────────────────────────┘                   │
 ├──────────────────┼────────────────────────────────────────────────────────────────────────┤
 │                  │ ┌───────────────────┬───────────────────┬────────────────────────┐     │
 │ #chunks <varint> │ │ c_i.mint <varint> │ c_i.maxt <varint> │ ref(c_i.data) <varint> │ ... │
 │                  │ └───────────────────┴───────────────────┴────────────────────────┘     │
 ├──────────────────┴────────────────────────────────────────────────────────────────────────┤
 │ CRC32 <4 byte>                                                                            │
 └───────────────────────────────────────────────────────────────────────────────────────────┘
```

The CRC checksum is calculated over the series contents of the index concatenated with the data of its chunks (with encoding byte, without length).


### Label Index

The label index indexes holds lists of possible values for label names.


### Postings

Postings are postings lists that map label pairs to series they occur in.


###
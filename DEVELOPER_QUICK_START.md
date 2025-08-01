# COBOL Examples - Developer Quick Start Guide

## Prerequisites Checklist

- [ ] Linux/Unix environment (Ubuntu 18.04+ recommended)
- [ ] GnuCOBOL compiler installed
- [ ] Basic understanding of COBOL syntax
- [ ] Text editor or IDE configured for COBOL

## 5-Minute Setup

### 1. Install GnuCOBOL (Ubuntu/Debian)
```bash
sudo apt-get update
sudo apt-get install build-essential libgmp-dev libdb-dev libncurses5-dev
wget https://sourceforge.net/projects/gnucobol/files/gnucobol/3.2/gnucobol-3.2.tar.xz
tar -xf gnucobol-3.2.tar.xz && cd gnucobol-3.2
./configure --with-xml2 --with-json
make && sudo make install && sudo ldconfig
```

### 2. Verify Installation
```bash
cobc --version
cobcrun --info
```

### 3. Test Basic Compilation
```bash
cd accept/
cobc -x accept.cbl
./accept
```

## Essential Examples to Try First

### 1. User Input (`accept/`)
```bash
cd accept/
cobc -x accept_from.cbl
./accept_from "test arg1" "test arg2"
```
**What it demonstrates**: Command-line arguments, environment variables, system data access

### 2. String Processing (`trim/` and `unstring/`)
```bash
cd trim/
cobc -x trim.cbl
./trim

cd ../unstring/
cobc -x unstring.cbl
./unstring
```
**What it demonstrates**: String manipulation, parsing, whitespace handling

### 3. Data Structures (`search/` and `redefines/`)
```bash
cd search/
cobc -x search.cbl
./search

cd ../redefines/
cobc -x redefines.cbl
./redefines
```
**What it demonstrates**: Table operations, memory overlays, data type conversion

## Advanced Examples (Require Additional Setup)

### XML/JSON Generation
```bash
# Requires libxml2-dev and libjson-c-dev
sudo apt-get install libxml2-dev libjson-c-dev
sudo ldconfig

cd xml_generate/
cobc -x xml_generate.cbl
./xml_generate

cd ../json_generate/
cobc -x json_generate.cbl
./json_generate
```

### Database Integration
```bash
# Requires PostgreSQL and esqlOC precompiler
sudo apt-get install postgresql unixodbc odbc-postgresql

# Setup test database (see sql/README.md for details)
cd sql/
# Follow database setup instructions
esqlOC -static -o generated_sql_ex.cbl sql_example.cbl
cobc -x -static -locsql generated_sql_ex.cbl
./generated_sql_ex
```

## Common Compilation Patterns

| Example Type | Compilation Command |
|--------------|-------------------|
| Basic COBOL | `cobc -x program.cbl` |
| With debugging | `cobc -x -g program.cbl` |
| Static linking | `cobc -x -static program.cbl` |
| SQL programs | `esqlOC -static -o output.cbl input.cbl && cobc -x -static -locsql output.cbl` |

## Troubleshooting Quick Fixes

**Compilation fails**: Check dependencies with `cobcrun --info`
**Library not found**: Run `sudo ldconfig`
**Permission denied**: Check file permissions with `ls -la`
**Screen mode issues**: Use explicit positioning: `display "text" at line 5 col 10`

## Next Steps

1. Read the full [COMPREHENSIVE_DOCUMENTATION.md](COMPREHENSIVE_DOCUMENTATION.md)
2. Explore specific directories based on your needs
3. Check individual README.md files in each directory
4. Try modifying examples to understand the concepts better

## Getting Help

- Check directory-specific README.md files
- Review error messages carefully
- Use `cobcrun --info` to verify library support
- Enable debugging with `cobc -x -g program.cbl`

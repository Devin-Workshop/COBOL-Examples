# COBOL Examples - Testing Guide

## Testing Strategy Overview

This guide provides comprehensive testing approaches for COBOL applications, covering unit testing, integration testing, performance testing, and automated testing strategies.

## Test Environment Setup

### Basic Testing Environment

```bash
# Create testing workspace
mkdir -p ~/cobol-testing
cd ~/cobol-testing

# Clone repository for testing
git clone https://github.com/Devin-Workshop/COBOL-Examples.git
cd COBOL-Examples

# Create test results directory
mkdir test-results
```

### Database Testing Setup

```bash
# Install PostgreSQL for testing
sudo apt-get install postgresql postgresql-contrib

# Create test database
sudo -u postgres createdb cobol_test_db

# Setup test user
sudo -u postgres psql -c "CREATE USER test_user WITH PASSWORD 'test_pass';"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE cobol_test_db TO test_user;"

# Load test data
sudo -u postgres psql cobol_test_db < sql/create_test_db.sql
```

## Unit Testing Approaches

### Testing Individual COBOL Programs

#### 1. Input/Output Testing

**Test Script for Accept Functionality**:
```bash
#!/bin/bash
# test_accept.sh

echo "Testing ACCEPT functionality..."

cd accept/
cobc -x accept_from.cbl

# Test command-line arguments
echo "Testing command-line arguments..."
output=$(echo "" | ./accept_from "arg1" "arg2" 2>&1)

if echo "$output" | grep -q "accept from command-line: arg1 arg2"; then
    echo "✓ PASS: Command-line argument processing"
else
    echo "✗ FAIL: Command-line argument processing"
    echo "Expected: 'accept from command-line: arg1 arg2'"
    echo "Got: $output"
fi

# Test environment variables
echo "Testing environment variables..."
export COB_TEST_ENV_KEY="test_value"
output=$(echo "" | ./accept_from 2>&1)

if echo "$output" | grep -q "accept from environment: test_value"; then
    echo "✓ PASS: Environment variable access"
else
    echo "✗ FAIL: Environment variable access"
fi

cd ..
```

#### 2. String Processing Testing

**Test Script for String Functions**:
```bash
#!/bin/bash
# test_string_processing.sh

echo "Testing string processing functionality..."

# Test TRIM function
cd trim/
cobc -x trim.cbl
output=$(echo "" | ./trim 2>&1)

if echo "$output" | grep -q "hello world"; then
    echo "✓ PASS: TRIM function basic operation"
else
    echo "✗ FAIL: TRIM function basic operation"
fi

cd ../unstring/
cobc -x unstring.cbl
output=$(echo "" | ./unstring 2>&1)

if echo "$output" | grep -q "PART1: Hello" && echo "$output" | grep -q "PART2: World"; then
    echo "✓ PASS: UNSTRING basic operation"
else
    echo "✗ FAIL: UNSTRING basic operation"
fi

cd ..
```

#### 3. Data Structure Testing

**Test Script for Search Operations**:
```bash
#!/bin/bash
# test_search.sh

echo "Testing search functionality..."

cd search/
cobc -x search.cbl

# Test binary search
echo -e "3\n2\n102\n499\n1" | ./search > test_output.txt

if grep -q "Record found" test_output.txt; then
    echo "✓ PASS: Binary search operation"
else
    echo "✗ FAIL: Binary search operation"
fi

# Test sequential search
if grep -q "ws-no-key-id: 0001" test_output.txt; then
    echo "✓ PASS: Sequential search operation"
else
    echo "✗ FAIL: Sequential search operation"
fi

rm -f test_output.txt
cd ..
```

### Automated Unit Test Framework

**Master Test Runner**:
```bash
#!/bin/bash
# run_all_tests.sh

set -e

TEST_DIR="test-results"
mkdir -p "$TEST_DIR"

echo "COBOL Examples Test Suite"
echo "========================="
echo "Start time: $(date)"
echo ""

TOTAL_TESTS=0
PASSED_TESTS=0
FAILED_TESTS=0

# Function to run individual test
run_test() {
    local test_name="$1"
    local test_script="$2"
    
    echo "Running $test_name..."
    TOTAL_TESTS=$((TOTAL_TESTS + 1))
    
    if bash "$test_script" > "$TEST_DIR/${test_name}.log" 2>&1; then
        echo "✓ PASS: $test_name"
        PASSED_TESTS=$((PASSED_TESTS + 1))
    else
        echo "✗ FAIL: $test_name"
        FAILED_TESTS=$((FAILED_TESTS + 1))
        echo "  See $TEST_DIR/${test_name}.log for details"
    fi
}

# Run all tests
run_test "accept_functionality" "test_accept.sh"
run_test "string_processing" "test_string_processing.sh"
run_test "search_operations" "test_search.sh"
run_test "xml_generation" "test_xml.sh"
run_test "json_generation" "test_json.sh"

# Summary
echo ""
echo "Test Summary"
echo "============"
echo "Total tests: $TOTAL_TESTS"
echo "Passed: $PASSED_TESTS"
echo "Failed: $FAILED_TESTS"
echo "Success rate: $(( PASSED_TESTS * 100 / TOTAL_TESTS ))%"
echo "End time: $(date)"

if [ $FAILED_TESTS -eq 0 ]; then
    echo "All tests passed!"
    exit 0
else
    echo "Some tests failed. Check logs in $TEST_DIR/"
    exit 1
fi
```

## Integration Testing

### Database Integration Testing

**Database Connection Test**:
```bash
#!/bin/bash
# test_database.sh

echo "Testing database integration..."

cd sql/

# Check if esqlOC is available
if ! command -v esqlOC &> /dev/null; then
    echo "✗ SKIP: esqlOC precompiler not found"
    exit 0
fi

# Precompile SQL program
if esqlOC -static -o generated_sql_ex.cbl sql_example.cbl; then
    echo "✓ PASS: SQL precompilation"
else
    echo "✗ FAIL: SQL precompilation"
    exit 1
fi

# Compile with SQL library
if cobc -x -static -locsql generated_sql_ex.cbl; then
    echo "✓ PASS: SQL program compilation"
else
    echo "✗ FAIL: SQL program compilation"
    exit 1
fi

# Test database connection (requires running PostgreSQL)
if timeout 10s echo "4" | ./generated_sql_ex > test_output.txt 2>&1; then
    if grep -q "Connected to database" test_output.txt; then
        echo "✓ PASS: Database connection"
    else
        echo "✗ FAIL: Database connection"
        echo "Output: $(cat test_output.txt)"
    fi
else
    echo "✗ SKIP: Database not available or connection timeout"
fi

rm -f test_output.txt generated_sql_ex
cd ..
```

### XML/JSON Integration Testing

**XML Generation Test**:
```bash
#!/bin/bash
# test_xml.sh

echo "Testing XML generation..."

cd xml_generate/

# Check if XML support is available
if ! cobcrun --info | grep -q "XML library"; then
    echo "✗ SKIP: XML library not available"
    exit 0
fi

# Compile XML program
if cobc -x xml_generate.cbl; then
    echo "✓ PASS: XML program compilation"
else
    echo "✗ FAIL: XML program compilation"
    exit 1
fi

# Test XML generation
if echo "" | ./xml_generate > test_output.txt 2>&1; then
    if grep -q "<?xml version" test_output.txt; then
        echo "✓ PASS: XML document generation"
    else
        echo "✗ FAIL: XML document generation"
        echo "Output: $(cat test_output.txt)"
    fi
else
    echo "✗ FAIL: XML program execution"
fi

rm -f test_output.txt xml_generate
cd ..
```

**JSON Generation Test**:
```bash
#!/bin/bash
# test_json.sh

echo "Testing JSON generation..."

cd json_generate/

# Check if JSON support is available
if ! cobcrun --info | grep -q "JSON library"; then
    echo "✗ SKIP: JSON library not available"
    exit 0
fi

# Compile JSON program
if cobc -x json_generate.cbl; then
    echo "✓ PASS: JSON program compilation"
else
    echo "✗ FAIL: JSON program compilation"
    exit 1
fi

# Test JSON generation
if echo "" | ./json_generate > test_output.txt 2>&1; then
    if grep -q '"ws-record"' test_output.txt; then
        echo "✓ PASS: JSON document generation"
    else
        echo "✗ FAIL: JSON document generation"
        echo "Output: $(cat test_output.txt)"
    fi
else
    echo "✗ FAIL: JSON program execution"
fi

rm -f test_output.txt json_generate
cd ..
```

## Performance Testing

### Benchmark Testing Framework

**Performance Test Script**:
```bash
#!/bin/bash
# performance_test.sh

echo "COBOL Performance Testing"
echo "========================"

# Function to measure execution time
measure_performance() {
    local program="$1"
    local test_name="$2"
    local iterations=100
    
    echo "Testing $test_name ($iterations iterations)..."
    
    start_time=$(date +%s.%N)
    for ((i=1; i<=iterations; i++)); do
        echo "" | ./"$program" > /dev/null 2>&1
    done
    end_time=$(date +%s.%N)
    
    total_time=$(echo "$end_time - $start_time" | bc)
    avg_time=$(echo "scale=6; $total_time / $iterations" | bc)
    
    echo "  Total time: ${total_time}s"
    echo "  Average time per execution: ${avg_time}s"
    echo "  Executions per second: $(echo "scale=2; $iterations / $total_time" | bc)"
    echo ""
}

# Compile programs for testing
cd accept/
cobc -x accept.cbl
measure_performance "accept" "Basic ACCEPT operations"
cd ..

cd trim/
cobc -x trim.cbl
measure_performance "trim" "String TRIM operations"
cd ..

cd search/
cobc -x search.cbl
# Note: This requires interactive input, so we'll skip automated performance testing
echo "Search operations require interactive input - manual testing recommended"
cd ..
```

### Memory Usage Testing

**Memory Profiling Script**:
```bash
#!/bin/bash
# memory_test.sh

echo "Memory Usage Testing"
echo "==================="

# Function to measure memory usage
measure_memory() {
    local program="$1"
    local test_name="$2"
    
    echo "Testing memory usage for $test_name..."
    
    # Use valgrind if available
    if command -v valgrind &> /dev/null; then
        echo "" | valgrind --tool=massif --massif-out-file=massif.out ./"$program" > /dev/null 2>&1
        if [ -f massif.out ]; then
            peak_memory=$(grep "mem_heap_B=" massif.out | sort -n | tail -1 | cut -d'=' -f2)
            echo "  Peak memory usage: $peak_memory bytes"
            rm -f massif.out
        fi
    else
        # Use /usr/bin/time for basic memory measurement
        /usr/bin/time -v echo "" | ./"$program" > /dev/null 2>&1
    fi
    echo ""
}

# Test memory usage for different programs
cd accept/
if [ -f accept ]; then
    measure_memory "accept" "ACCEPT operations"
fi
cd ..

cd trim/
if [ -f trim ]; then
    measure_memory "trim" "String processing"
fi
cd ..
```

## Test Data Management

### Test Data Generation

**Sample Data Generator**:
```bash
#!/bin/bash
# generate_test_data.sh

echo "Generating test data..."

# Create test input files
mkdir -p test-data

# Generate customer test data
cat > test-data/customers.txt << 'EOF'
00001,John,Doe,5551234567,123 Main St
00002,Jane,Smith,5559876543,456 Oak Ave
00003,Bob,Johnson,5555555555,789 Pine Rd
00004,Alice,Brown,5551111111,321 Elm St
00005,Charlie,Davis,5552222222,654 Maple Dr
EOF

# Generate transaction test data
cat > test-data/transactions.txt << 'EOF'
T001,00001,2023-01-15,100.50,DEPOSIT
T002,00001,2023-01-16,25.00,WITHDRAWAL
T003,00002,2023-01-15,500.00,DEPOSIT
T004,00003,2023-01-17,75.25,WITHDRAWAL
T005,00002,2023-01-18,200.00,DEPOSIT
EOF

# Generate configuration test data
cat > test-data/config.txt << 'EOF'
DATABASE_HOST=localhost
DATABASE_PORT=5432
DATABASE_NAME=cobol_test_db
DATABASE_USER=test_user
DATABASE_PASS=test_pass
XML_OUTPUT_DIR=/tmp/xml_output
JSON_OUTPUT_DIR=/tmp/json_output
EOF

echo "Test data generated in test-data/ directory"
```

### Test Data Validation

**Data Validation Script**:
```bash
#!/bin/bash
# validate_test_data.sh

echo "Validating test data..."

# Validate customer data format
if [ -f test-data/customers.txt ]; then
    echo "Validating customer data..."
    while IFS=',' read -r id name surname phone address; do
        # Check ID format (5 digits)
        if [[ ! $id =~ ^[0-9]{5}$ ]]; then
            echo "  Warning: Invalid customer ID format: $id"
        fi
        
        # Check phone format
        if [[ ! $phone =~ ^[0-9]{10}$ ]]; then
            echo "  Warning: Invalid phone format: $phone"
        fi
    done < test-data/customers.txt
    echo "  Customer data validation complete"
fi

# Validate transaction data format
if [ -f test-data/transactions.txt ]; then
    echo "Validating transaction data..."
    while IFS=',' read -r trans_id cust_id date amount type; do
        # Check transaction ID format
        if [[ ! $trans_id =~ ^T[0-9]{3}$ ]]; then
            echo "  Warning: Invalid transaction ID format: $trans_id"
        fi
        
        # Check amount format
        if [[ ! $amount =~ ^[0-9]+\.[0-9]{2}$ ]]; then
            echo "  Warning: Invalid amount format: $amount"
        fi
        
        # Check transaction type
        if [[ ! $type =~ ^(DEPOSIT|WITHDRAWAL)$ ]]; then
            echo "  Warning: Invalid transaction type: $type"
        fi
    done < test-data/transactions.txt
    echo "  Transaction data validation complete"
fi

echo "Test data validation complete"
```

## Continuous Integration Testing

### CI Pipeline Configuration

**GitHub Actions Workflow** (`.github/workflows/test.yml`):
```yaml
name: COBOL Examples CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:13
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: cobol_test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Install dependencies
      run: |
        sudo apt-get update
        sudo apt-get install -y build-essential libgmp-dev libdb-dev libncurses5-dev
        sudo apt-get install -y libxml2-dev libjson-c-dev
        sudo apt-get install -y postgresql-client unixodbc odbc-postgresql
    
    - name: Install GnuCOBOL
      run: |
        wget https://sourceforge.net/projects/gnucobol/files/gnucobol/3.2/gnucobol-3.2.tar.xz
        tar -xf gnucobol-3.2.tar.xz
        cd gnucobol-3.2
        ./configure --with-xml2 --with-json
        make
        sudo make install
        sudo ldconfig
    
    - name: Verify installation
      run: |
        cobc --version
        cobcrun --info
    
    - name: Setup test database
      run: |
        PGPASSWORD=postgres psql -h localhost -U postgres -d cobol_test_db -f sql/create_test_db.sql
      env:
        PGPASSWORD: postgres
    
    - name: Run tests
      run: |
        chmod +x run_all_tests.sh
        ./run_all_tests.sh
    
    - name: Upload test results
      uses: actions/upload-artifact@v3
      if: always()
      with:
        name: test-results
        path: test-results/
```

### Quality Gates

**Code Quality Checks**:
```bash
#!/bin/bash
# quality_check.sh

echo "Running code quality checks..."

# Check for common COBOL coding standards
echo "Checking coding standards..."

# Check for proper program structure
find . -name "*.cbl" -exec grep -L "identification division" {} \; | while read file; do
    echo "Warning: $file missing identification division"
done

# Check for proper data division structure
find . -name "*.cbl" -exec grep -L "data division" {} \; | while read file; do
    echo "Warning: $file missing data division"
done

# Check for proper procedure division structure
find . -name "*.cbl" -exec grep -L "procedure division" {} \; | while read file; do
    echo "Warning: $file missing procedure division"
done

# Check for consistent indentation (basic check)
find . -name "*.cbl" | while read file; do
    if grep -q "^[^ ].*[a-zA-Z]" "$file"; then
        # Check if there are lines that should be indented but aren't
        echo "Info: $file may have indentation issues"
    fi
done

echo "Code quality check complete"
```

## Test Reporting

### Test Report Generation

**HTML Test Report Generator**:
```bash
#!/bin/bash
# generate_report.sh

echo "Generating test report..."

cat > test-results/report.html << 'EOF'
<!DOCTYPE html>
<html>
<head>
    <title>COBOL Examples Test Report</title>
    <style>
        body { font-family: Arial, sans-serif; margin: 20px; }
        .pass { color: green; }
        .fail { color: red; }
        .skip { color: orange; }
        table { border-collapse: collapse; width: 100%; }
        th, td { border: 1px solid #ddd; padding: 8px; text-align: left; }
        th { background-color: #f2f2f2; }
    </style>
</head>
<body>
    <h1>COBOL Examples Test Report</h1>
    <p>Generated on: $(date)</p>
    
    <h2>Test Summary</h2>
    <table>
        <tr><th>Test Suite</th><th>Status</th><th>Details</th></tr>
EOF

# Process test results
for log_file in test-results/*.log; do
    if [ -f "$log_file" ]; then
        test_name=$(basename "$log_file" .log)
        if grep -q "PASS" "$log_file"; then
            status="<span class='pass'>PASS</span>"
        elif grep -q "FAIL" "$log_file"; then
            status="<span class='fail'>FAIL</span>"
        else
            status="<span class='skip'>SKIP</span>"
        fi
        
        echo "        <tr><td>$test_name</td><td>$status</td><td><a href='$log_file'>View Log</a></td></tr>" >> test-results/report.html
    fi
done

cat >> test-results/report.html << 'EOF'
    </table>
    
    <h2>Environment Information</h2>
    <pre>
EOF

cobc --version >> test-results/report.html 2>&1
cobcrun --info >> test-results/report.html 2>&1

cat >> test-results/report.html << 'EOF'
    </pre>
</body>
</html>
EOF

echo "Test report generated: test-results/report.html"
```

This comprehensive testing guide provides the foundation for ensuring quality and reliability in COBOL applications, from basic unit tests to complex integration scenarios and automated CI/CD pipelines.

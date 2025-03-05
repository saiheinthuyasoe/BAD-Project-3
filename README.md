# AWS Project Implementation Guide

Based on the project description, I'll provide a step-by-step guide to implement the system according to the architecture diagram. This guide will focus on using free tier options where possible.

## 1. Setting Up MySQL RDS

### Create a MySQL RDS Instance (Free Tier)

1. Log in to the AWS Management Console
2. Navigate to RDS service
3. Click "Create database"
4. Choose the following settings:
   - Select "Standard create"
   - Engine type: MySQL
   - Version: MySQL 8.0.28 (or latest available in free tier)
   - Templates: Free tier
   - DB instance identifier: employee-db (or your preferred name)
   - Master username: admin (or your preferred username)
   - Master password: (create a secure password)
   - DB instance class: db.t2.micro or db.t3.micro (free tier eligible)
   - Storage: General Purpose SSD, 20 GB (free tier limit)
   - Enable storage autoscaling: No (to avoid unexpected charges)
   - VPC: Default VPC
   - Public access: Yes (for this project only, normally not recommended)
   - VPC security group: Create new
   - Availability Zone: No preference
   - Database authentication: Password authentication
   - Initial database name: employees
   - Backup retention period: 1 day (minimum)
   - Disable automated backups: No
   - Encryption: Disable encryption (for free tier)
   - Monitoring: Disable enhanced monitoring (for free tier)
   - Maintenance: Enable auto minor version upgrade

5. Click "Create database"

### Configure Security Group

1. Go to EC2 > Security Groups
2. Find the security group associated with your RDS instance
3. Edit inbound rules
4. Add a rule:
   - Type: MySQL/Aurora
   - Protocol: TCP
   - Port Range: 3306
   - Source: 0.0.0.0/0 (for testing only - in production you would restrict this)
5. Save rules

### Load the Sample Database

1. Clone the test_db repository on your local machine:

```bash
git clone https://github.com/datacharmer/test_db
cd test_db
```

2. Install MySQL Shell if you don't have it:
   - Download from: https://dev.mysql.com/doc/mysql-shell/8.4/en/mysql-shell-install.html
   - For Windows, use the MSI installer

3. Load the database (replace placeholders with your actual values):

```bash
mysql -h your-rds-endpoint.rds.amazonaws.com -u admin -pyourpassword < employees.sql
```

4. Verify the database was loaded correctly:

```bash
mysql -h your-rds-endpoint.rds.amazonaws.com -u admin -pyourpassword -e "USE employees; SELECT COUNT(*) FROM employees;"
```

## 2. Setting Up Amazon ElastiCache

### Create an ElastiCache Cluster (Redis)

1. Go to ElastiCache in AWS Console
2. Click "Create"
3. Choose Redis
4. Configure:
   - Name: employees-cache
   - Engine version: Latest compatible version
   - Port: 6379 (default)
   - Parameter group: default
   - Node type: cache.t2.micro (free tier eligible)
   - Number of replicas: 0 (to stay within free tier)
   - Multi-AZ: No (to stay within free tier)
   - Subnet group: Create new or use default
   - Security: Use the same security group as your RDS instance
5. Create the cache cluster

## 3. Creating Lambda Functions for CRUD Operations

### Create IAM Role for Lambda

1. Go to IAM in AWS Console
2. Create a new role with the following permissions:
   - AWSLambdaBasicExecutionRole
   - AmazonRDSDataFullAccess
   - AmazonElastiCacheFullAccess
   - AmazonS3ReadOnlyAccess

### Create Lambda Functions

Create four Lambda functions for CRUD operations. Here's an example for one function using Node.js:

1. Go to Lambda in AWS Console
2. Click "Create function"
3. Choose "Author from scratch"
4. Configure:
   - Function name: EmployeeGet (or similar)
   - Runtime: Node.js 14.x
   - Architecture: x86_64
   - Permissions: Use the IAM role created earlier
5. Create the function

Here's a sample code for the Get function:

```javascript:c:\Users\saihe\OneDrive\Documents\BAD Project 2\lambda\getEmployee.js
const mysql = require('mysql2/promise');
const redis = require('redis');
const { promisify } = require('util');

// Database configuration
const dbConfig = {
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: 'employees'
};

// Redis configuration
const redisConfig = {
  host: process.env.REDIS_HOST,
  port: 6379
};

exports.handler = async (event) => {
  let connection;
  let redisClient;
  
  try {
    // Parse the employee ID from the event
    const empNo = event.pathParameters?.empNo || event.queryStringParameters?.empNo;
    
    // If no employee ID is provided, return top 10 employees with high salary
    if (!empNo) {
      // Try to get from cache first
      redisClient = redis.createClient(redisConfig);
      const getAsync = promisify(redisClient.get).bind(redisClient);
      
      // Cache key for the top 10 query
      const cacheKey = 'top10_employees';
      
      try {
        // Try to get from cache
        const cachedData = await getAsync(cacheKey);
        if (cachedData) {
          console.log('Data retrieved from cache');
          return {
            statusCode: 200,
            headers: {
              'Content-Type': 'application/json',
              'Access-Control-Allow-Origin': '*'
            },
            body: cachedData
          };
        }
      } catch (cacheError) {
        console.error('Cache error:', cacheError);
        // Continue to database if cache fails
      }
      
      // Connect to the database
      connection = await mysql.createConnection(dbConfig);
      
      // Execute the query for top 10 employees
      const [rows] = await connection.execute(`
        SELECT e.emp_no, e.first_name, e.last_name, d.dept_name, MAX(s.salary) AS max_salary 
        FROM employees e 
        JOIN dept_emp de ON e.emp_no = de.emp_no 
        JOIN departments d ON de.dept_no = d.dept_no 
        JOIN (SELECT emp_no, salary FROM salaries WHERE to_date = '9999-01-01') s 
        ON e.emp_no = s.emp_no 
        WHERE s.salary > (SELECT AVG(salary) FROM salaries) 
        GROUP BY e.emp_no, e.first_name, e.last_name, d.dept_name 
        ORDER BY max_salary DESC 
        LIMIT 10
      `);
      
      // Store in cache for future requests (with 5 minute expiration)
      if (redisClient) {
        const resultJson = JSON.stringify(rows);
        redisClient.setex(cacheKey, 300, resultJson);
      }
      
      return {
        statusCode: 200,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*'
        },
        body: JSON.stringify(rows)
      };
    } else {
      // Get specific employee by ID
      connection = await mysql.createConnection(dbConfig);
      
      // Get employee details
      const [employee] = await connection.execute(
        'SELECT * FROM employees WHERE emp_no = ?',
        [empNo]
      );
      
      if (employee.length === 0) {
        return {
          statusCode: 404,
          headers: {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*'
          },
          body: JSON.stringify({ message: 'Employee not found' })
        };
      }
      
      // Get current salary
      const [salary] = await connection.execute(
        'SELECT salary FROM salaries WHERE emp_no = ? AND to_date = "9999-01-01"',
        [empNo]
      );
      
      // Get department
      const [department] = await connection.execute(`
        SELECT d.dept_name 
        FROM dept_emp de 
        JOIN departments d ON de.dept_no = d.dept_no 
        WHERE de.emp_no = ? AND de.to_date = "9999-01-01"
      `, [empNo]);
      
      const result = {
        ...employee[0],
        salary: salary.length > 0 ? salary[0].salary : null,
        department: department.length > 0 ? department[0].dept_name : null
      };
      
      return {
        statusCode: 200,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*'
        },
        body: JSON.stringify(result)
      };
    }
  } catch (error) {
    console.error('Error:', error);
    return {
      statusCode: 500,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({ message: 'Internal server error', error: error.message })
    };
  } finally {
    // Close connections
    if (connection) await connection.end();
    if (redisClient) redisClient.quit();
  }
};
```

Similarly, create the other CRUD functions:

```javascript:c:\Users\saihe\OneDrive\Documents\BAD Project 2\lambda\createEmployee.js
const mysql = require('mysql2/promise');
const redis = require('redis');

// Database configuration
const dbConfig = {
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: 'employees'
};

// Redis configuration
const redisConfig = {
  host: process.env.REDIS_HOST,
  port: 6379
};

exports.handler = async (event) => {
  let connection;
  let redisClient;
  
  try {
    // Parse the request body
    const employee = JSON.parse(event.body);
    
    // Validate required fields
    if (!employee.first_name || !employee.last_name || !employee.birth_date || 
        !employee.gender || !employee.hire_date || !employee.salary || !employee.dept_no) {
      return {
        statusCode: 400,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*'
        },
        body: JSON.stringify({ message: 'Missing required fields' })
      };
    }
    
    // Connect to the database
    connection = await mysql.createConnection(dbConfig);
    
    // Start transaction
    await connection.beginTransaction();
    
    // Get the next employee number
    const [maxEmpNo] = await connection.execute('SELECT MAX(emp_no) as max_emp_no FROM employees');
    const newEmpNo = maxEmpNo[0].max_emp_no + 1;
    
    // Insert into employees table
    await connection.execute(
      'INSERT INTO employees (emp_no, birth_date, first_name, last_name, gender, hire_date) VALUES (?, ?, ?, ?, ?, ?)',
      [newEmpNo, employee.birth_date, employee.first_name, employee.last_name, employee.gender, employee.hire_date]
    );
    
    // Insert into salaries table
    await connection.execute(
      'INSERT INTO salaries (emp_no, salary, from_date, to_date) VALUES (?, ?, ?, ?)',
      [newEmpNo, employee.salary, employee.hire_date, '9999-01-01']
    );
    
    // Insert into dept_emp table
    await connection.execute(
      'INSERT INTO dept_emp (emp_no, dept_no, from_date, to_date) VALUES (?, ?, ?, ?)',
      [newEmpNo, employee.dept_no, employee.hire_date, '9999-01-01']
    );
    
    // Commit transaction
    await connection.commit();
    
    // Invalidate cache
    redisClient = redis.createClient(redisConfig);
    redisClient.del('top10_employees');
    
    return {
      statusCode: 201,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({ 
        message: 'Employee created successfully', 
        emp_no: newEmpNo 
      })
    };
  } catch (error) {
    console.error('Error:', error);
    
    // Rollback transaction if there was an error
    if (connection) {
      try {
        await connection.rollback();
      } catch (rollbackError) {
        console.error('Rollback error:', rollbackError);
      }
    }
    
    return {
      statusCode: 500,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({ message: 'Internal server error', error: error.message })
    };
  } finally {
    // Close connections
    if (connection) await connection.end();
    if (redisClient) redisClient.quit();
  }
};
```

```javascript:c:\Users\saihe\OneDrive\Documents\BAD Project 2\lambda\updateEmployee.js
const mysql = require('mysql2/promise');
const redis = require('redis');

// Database configuration
const dbConfig = {
  host: process.env.DB_HOST,
  user: process.env.DB_USER,
  password: process.env.DB_PASSWORD,
  database: 'employees'
};

// Redis configuration
const redisConfig = {
  host: process.env.REDIS_HOST,
  port: 6379
};

exports.handler = async (event) => {
  let connection;
  let redisClient;
  
  try {
    // Parse the employee ID and request body
    const empNo = event.pathParameters?.empNo || event.queryStringParameters?.empNo;
    const updates = JSON.parse(event.body);
    
    if (!empNo) {
      return {
        statusCode: 400,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*'
        },
        body: JSON.stringify({ message: 'Employee ID is required' })
      };
    }
    
    // Connect to the database
    connection = await mysql.createConnection(dbConfig);
    
    // Start transaction
    await connection.beginTransaction();
    
    // Check if employee exists
    const [employee] = await connection.execute(
      'SELECT * FROM employees WHERE emp_no = ?',
      [empNo]
    );
    
    if (employee.length === 0) {
      return {
        statusCode: 404,
        headers: {
          'Content-Type': 'application/json',
          'Access-Control-Allow-Origin': '*'
        },
        body: JSON.stringify({ message: 'Employee not found' })
      };
    }
    
    // Update employee basic info if provided
    if (updates.first_name || updates.last_name || updates.birth_date || updates.gender || updates.hire_date) {
      let updateFields = [];
      let updateValues = [];
      
      if (updates.first_name) {
        updateFields.push('first_name = ?');
        updateValues.push(updates.first_name);
      }
      
      if (updates.last_name) {
        updateFields.push('last_name = ?');
        updateValues.push(updates.last_name);
      }
      
      if (updates.birth_date) {
        updateFields.push('birth_date = ?');
        

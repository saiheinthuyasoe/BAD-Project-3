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

I'll continue with the implementation guide for the project, focusing on completing the Lambda functions and setting up the remaining components.

## 3. Creating Lambda Functions for CRUD Operations (continued)

Let's complete the update and delete Lambda functions:

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
        updateValues.push(updates.birth_date);
      }
      
      if (updates.gender) {
        updateFields.push('gender = ?');
        updateValues.push(updates.gender);
      }
      
      if (updates.hire_date) {
        updateFields.push('hire_date = ?');
        updateValues.push(updates.hire_date);
      }
      
      if (updateFields.length > 0) {
        // Add emp_no to values array for WHERE clause
        updateValues.push(empNo);
        
        await connection.execute(
          `UPDATE employees SET ${updateFields.join(', ')} WHERE emp_no = ?`,
          updateValues
        );
      }
    }
    
    // Update salary if provided
    if (updates.salary) {
      const today = new Date().toISOString().split('T')[0];
      
      // End current salary record
      await connection.execute(
        'UPDATE salaries SET to_date = ? WHERE emp_no = ? AND to_date = "9999-01-01"',
        [today, empNo]
      );
      
      // Insert new salary record
      await connection.execute(
        'INSERT INTO salaries (emp_no, salary, from_date, to_date) VALUES (?, ?, ?, ?)',
        [empNo, updates.salary, today, '9999-01-01']
      );
    }
    
    // Update department if provided
    if (updates.dept_no) {
      const today = new Date().toISOString().split('T')[0];
      
      // End current department assignment
      await connection.execute(
        'UPDATE dept_emp SET to_date = ? WHERE emp_no = ? AND to_date = "9999-01-01"',
        [today, empNo]
      );
      
      // Insert new department assignment
      await connection.execute(
        'INSERT INTO dept_emp (emp_no, dept_no, from_date, to_date) VALUES (?, ?, ?, ?)',
        [empNo, updates.dept_no, today, '9999-01-01']
      );
    }
    
    // Commit transaction
    await connection.commit();
    
    // Invalidate cache
    redisClient = redis.createClient(redisConfig);
    redisClient.del('top10_employees');
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({ 
        message: 'Employee updated successfully', 
        emp_no: empNo 
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

```javascript:c:\Users\saihe\OneDrive\Documents\BAD Project 2\lambda\deleteEmployee.js
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
    // Parse the employee ID
    const empNo = event.pathParameters?.empNo || event.queryStringParameters?.empNo;
    
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
    
    // Delete from salaries
    await connection.execute(
      'DELETE FROM salaries WHERE emp_no = ?',
      [empNo]
    );
    
    // Delete from dept_emp
    await connection.execute(
      'DELETE FROM dept_emp WHERE emp_no = ?',
      [empNo]
    );
    
    // Delete from dept_manager if applicable
    await connection.execute(
      'DELETE FROM dept_manager WHERE emp_no = ?',
      [empNo]
    );
    
    // Delete from titles if applicable
    await connection.execute(
      'DELETE FROM titles WHERE emp_no = ?',
      [empNo]
    );
    
    // Finally delete from employees
    await connection.execute(
      'DELETE FROM employees WHERE emp_no = ?',
      [empNo]
    );
    
    // Commit transaction
    await connection.commit();
    
    // Invalidate cache
    redisClient = redis.createClient(redisConfig);
    redisClient.del('top10_employees');
    
    return {
      statusCode: 200,
      headers: {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
      },
      body: JSON.stringify({ 
        message: 'Employee deleted successfully', 
        emp_no: empNo 
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

### Deploy Lambda Functions

For each Lambda function:

1. Create a deployment package:
   - Install dependencies:
   ```bash
   cd c:\Users\saihe\OneDrive\Documents\BAD Project 2\lambda
   npm init -y
   npm install mysql2 redis
   ```
   - Zip the function and node_modules:
   ```bash
   powershell Compress-Archive -Path .\getEmployee.js, .\node_modules -DestinationPath .\getEmployee.zip
   powershell Compress-Archive -Path .\createEmployee.js, .\node_modules -DestinationPath .\createEmployee.zip
   powershell Compress-Archive -Path .\updateEmployee.js, .\node_modules -DestinationPath .\updateEmployee.zip
   powershell Compress-Archive -Path .\deleteEmployee.js, .\node_modules -DestinationPath .\deleteEmployee.zip
   ```

2. Upload each zip file to the corresponding Lambda function
3. Configure environment variables for each Lambda:
   - DB_HOST: Your RDS endpoint
   - DB_USER: Your database username
   - DB_PASSWORD: Your database password
   - REDIS_HOST: Your ElastiCache endpoint

## 4. Setting Up API Gateway

1. Go to API Gateway in AWS Console
2. Create a new REST API
3. Create resources and methods:

   - `/employees` (GET) - List top 10 employees
   - `/employees/{empNo}` (GET) - Get employee by ID
   - `/employees` (POST) - Create employee
   - `/employees/{empNo}` (PUT) - Update employee
   - `/employees/{empNo}` (DELETE) - Delete employee

4. For each method:
   - Integration type: Lambda Function
   - Lambda Function: Select the corresponding function
   - Use Lambda Proxy integration: Yes

5. Enable CORS for all methods:
   - Access-Control-Allow-Origin: '*'
   - Access-Control-Allow-Headers: 'Content-Type,X-Amz-Date,Authorization,X-Api-Key,X-Amz-Security-Token'
   - Access-Control-Allow-Methods: 'GET,POST,PUT,DELETE,OPTIONS'

6. Deploy the API:
   - Create a new stage (e.g., "prod")
   - Note the Invoke URL

## 5. Creating the S3 Static Website

### Create S3 Bucket

1. Go to S3 in AWS Console
2. Create a new bucket:
   - Name: choose a globally unique name
   - Region: same as your other resources
   - Block all public access: Uncheck
3. Enable static website hosting:
   - Properties > Static website hosting > Enable
   - Index document: index.html
   - Error document: error.html
4. Add bucket policy to make content public:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "PublicReadGetObject",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:GetObject",
         "Resource": "arn:aws:s3:::your-bucket-name/*"
       }
     ]
   }
   ```

### Create Website Files

Create the following files for your website:

```html:c:\Users\saihe\OneDrive\Documents\BAD Project 2\website\index.html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Employee Management System</title>
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.3.0/dist/css/bootstrap.min.css" rel="stylesheet">
    <link rel="stylesheet" href="styles.css">
</head>
<body>
    <nav class="navbar navbar-expand-lg navbar-dark bg-dark">
        <div class="container">
            <a class="navbar-brand" href="/">Employee Management System</a>
            <button class="navbar-toggler" type="button" data-bs-toggle="collapse" data-bs-target="#navbarNav">
                <span class="navbar-toggler-icon"></span>
            </button>
            <div class="collapse navbar-collapse" id="navbarNav">
                <ul class="navbar-nav ms-auto">
                    <li class="nav-item">
                        <a class="nav-link active" href="/">Home</a>
                    </li>
                    <li class="nav-item">
                        <a class="nav-link" href="/about.html">About</a>
                    </li>
                </ul>
            </div>
        </div>
    </nav>

    <div class="container mt-4">
        <h1>Top 10 Highest Paid Current Employees</h1>
        <div id="loading" class="text-center my-5">
            <div class="spinner-border text-primary" role="status">
                <span class="visually-hidden">Loading...</span>
            </div>
            <p class="mt-2">Loading data...</p>
        </div>
        <div id="query-time" class="alert alert-info d-none">
            Query time: <span id="time-value">0</span> ms
        </div>
        <table id="employees-table" class="table table-striped d-none">
            <thead>
                <tr>
                    <th>Employee ID</th>
                    <th>First Name</th>
                    <th>Last Name</th>
                    <th>Department</th>
                    <th>Salary</th>
                    <th>Actions</th>
                </tr>
            </thead>
            <tbody id="employees-data">
                <!-- Data will be loaded here -->
            </tbody>
        </table>

        <div class="mt-4">
            <button type="button" class="btn btn-primary" data-bs-toggle="modal" data-bs-target="#addEmployeeModal">
                Add New Employee
            </button>
        </div>
    </div>

    <!-- Add Employee Modal -->
    <div class="modal fade" id="addEmployeeModal" tabindex="-1" aria-hidden="true">
        <div class="modal-dialog">
            <div class="modal-content">
                <div class="modal-header">
                    <h5 class="modal-title

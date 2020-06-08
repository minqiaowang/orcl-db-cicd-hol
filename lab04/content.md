# Oracle Autonomous Database for Python

## Introduction

Our application data is stored in an Oracle ATP, that runs as an Autonomous Database Cloud Service on the OCI.

We want our Python web micro service to connect to our ATP, retrieve information about the employees in Human Resources (HR) sample schema, and calculate a promotion or a salary increase.

## Step 1: Download the ATP Client Credentials

Connect to your Compute Instance with Remote Desktop. Open a browser and connect to the OCI console. Click on hamburger menu ≡, then **Autonomous Transaction Processing** under Databases. Make sure you choose the correct Region and Compartment. Click **[Your Initials]ATP** for details. Click **Database Connection** and Click **Download Wallet**. You need to create a password for the wallet. For example: `WelCom3#2020_`

Click **Download** and **Save File**.


Back to the terminal connecting the VM. You can verify that a zip file has been downloaded in the directory.

````
$ ls /home/oracle/Downloads
Wallet_[Your Initials]ATP.zip
````

A version of 18.5 Oracle instant client has already been installed in this VM. Now you need to exit the **oracle** user and work as the **opc** user. Unzip the file to the directory of the oracle instant client.

````
$ sudo unzip -o -d /usr/lib/oracle/18.5/client64/lib/network/admin/ /home/oracle/Downloads/Wallet_[Your Initials]ATP.zip 
````
Back to the **oracle** user again.

````
sudo su - oracle
cd ~/orcl-ws-cicd
. ./orclvenv/bin/activate
````

Test the ATP connection with sqlplus.

````
$ export PATH=/usr/lib/oracle/18.5/client64/bin:$PATH
$ export LD_LIBRARY_PATH=/usr/lib/oracle/18.5/client64/lib
$ sqlplus admin/WelCom3#2020_@[Your Initials]ATP_TP

SQL*Plus: Release 18.0.0.0.0 - Production on Fri Jun 5 05:02:37 2020
Version 18.5.0.0.0

Copyright (c) 1982, 2018, Oracle.  All rights reserved.


Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.5.0.0.0

SQL> 
````

## Step 2: Prepare Sample Schema and Data

In the SQLPLUS, connect to **admin** user as previous step, create the **HR** user.

````
CREATE USER hr IDENTIFIED BY WelCom3#2020_;
GRANT CREATE SESSION TO hr;
GRANT DWROLE TO hr;
GRANT UNLIMITED TABLESPACE TO hr;
````

Create the Cloud Object Storage credential. We will use the pre-authenticated URI to import the sample data, but it's still need to supply a credential parameter. However, credentials for a pre-authenticated URL are ignored (and the supplied credentials do not need to be valid). So don't worry about the username and password parameter values to create the credential. 

````
connect hr/WelCom3#2020_@[Your Initials]ATP_TP;
BEGIN
  DBMS_CLOUD.CREATE_CREDENTIAL(
    credential_name => 'DEF_CRED_NAME',
    username => 'atp_user@example.com',
    password => 'password'
  );
END;
/
exit;
````

Import data into HR schema.

````
$ impdp hr/WelCom3#2020_@[Your Initials]ATP_TP directory=data_pump_dir credential=def_cred_name dumpfile= https://objectstorage.ap-seoul-1.oraclecloud.com/p/dqWb29uU2L1hwltBvjP58EHl25Pi90vBevHTR93F9tw/n/oraclepartnersas/b/DB19c/o/hr.dmp     
````

Test HR schema.

```
$ sqlplus hr/WelCom3#2020_@[Your Initials]ATP_TP

SQL*Plus: Release 18.0.0.0.0 - Production on Fri Jun 5 06:12:10 2020
Version 18.5.0.0.0

Copyright (c) 1982, 2018, Oracle.  All rights reserved.

Last Successful login time: Fri Jun 05 2020 06:09:04 +00:00

Connected to:
Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.5.0.0.0

SQL> select table_name from user_tables;

TABLE_NAME
--------------------------------------------------------------------------------
COUNTRIES
JOB_HISTORY
LOCATIONS
JOBS
DEPARTMENTS
REGIONS
EMPLOYEES

7 rows selected.

SQL> exit
Disconnected from Oracle Database 19c Enterprise Edition Release 19.0.0.0.0 - Production
Version 19.5.0.0.0
```



## Step 3: Connect to ATP from Python

Back on our development environment, let's connect our Python microservice to our Oracle Autonomous Database. This enhancement requires [cx_Oracle](https://oracle.github.io/python-cx_Oracle/) extension module.

````
pip install cx_Oracle
pip freeze > requirements.pip
````

Edit promotion.py, and add a new line to import cx_Oracle extension module.

````
import cx_Oracle
````

Add a new function with routing to test the database connection.

````
@app.route('/conn')
def conn():
    return str(connection.version)
````

Finally, add the connection code to the '\_\_main__' section.

````
    DBUSER = 'hr'
    DBPASS = 'WelCom3#2020_'
    DBSERV = '[Your Initials]ATP_TP'
    conn_string = DBUSER + '/' + DBPASS + '@' + DBSERV
    connection = cx_Oracle.connect(conn_string)
    run(app, host='0.0.0.0', port=8080)
    connection.close()
````

Just a quick check, your promotion.py should look like this, make sure the value of DBSERV is changed to your own ATP alias. 

````
"""
Simple Python application to show CI/CD capabilities.
"""

from bottle import Bottle, run
import cx_Oracle

app = Bottle()


@app.route('/addition/<salary>/<amount>')
def addition(salary, amount):
    return str(int(salary) + int(amount))


@app.route('/increment/<salary>/<percentage>')
def increment(salary, percentage):
    return str(int(salary) * (1 + int(percentage)/100))


@app.route('/decrease/<salary>/<amount>')
def decrease(salary, amount):
    return str(int(salary) - int(amount))


@app.route('/conn')
def conn():
    return str(connection.version)


if __name__ == '__main__':
    DBUSER = 'hr'
    DBPASS = 'WelCom3#2020_'
    DBSERV = '[Your Initials]ATP_TP'
    conn_string = DBUSER + '/' + DBPASS + '@' + DBSERV
    connection = cx_Oracle.connect(conn_string)
    run(app, host='0.0.0.0', port=8080)
    connection.close()
````

Commit and push the changes to the master branch on code repository.

````
git commit -a -m "Add database connection"
git push
````

Verify build is successful. 



## Step 4: Test ATP Connection

On our development environment, test the application

````
python3 promotion.py 
Bottle v0.12.18 server starting up (using WSGIRefServer())...
Listening on http://0.0.0.0:8080/
Hit Ctrl-C to quit.
````

Use the web browser on your laptop to open [http://localhost:8080/conn](http://localhost:8080/conn). The response is '19.5.0.0.0'. Your Python web micro service application is connected to your ATP. Press Ctrl-C to stop the application.

Edit `test_promotion.py`, and add a unit test for ATP connection. This is how it has to be, make sure the value of DBSERV is changed to your own ATP alias.

````
"""
Unit tests for simple Python application
"""

import promotion
import pytest
from webtest import TestApp
import cx_Oracle


class TestPromotion:

    def test_addition(self):
        assert '1200' == promotion.addition(1150, 50)

    def test_increment(self):
        assert '1250.0' == promotion.increment(1000, 25)

    def test_decrease(self):
        assert '970' == promotion.decrease(1150, 180)


@pytest.fixture
def application():
    test_app = TestApp(promotion.app)
    return test_app


def test_response_shold_be_ok(application):
    response = application.get('/addition/1000/200')
    assert response.status == "200 OK"


def test_addition(application):
    response = application.get('/addition/1000/200')
    assert b'1200' == response.body


@pytest.fixture(scope='session')
def test_connection():
    DBUSER = 'hr'
    DBPASS = 'WelCom3#2020_'
    DBSERV = '[Your Initials]ATP_TP'
    conn_string = DBUSER + '/' + DBPASS + '@' + DBSERV
    connection = cx_Oracle.connect(conn_string)
    response = connection.version
    assert response == '19.5.0.0.0'
    connection.close()
````

We added a line at the top to import `cx_Oracle` extension module, and a fixture that will only be run once for the entire test session.

Commit and push the changes to the master branch on code repository.

````
git commit -a -m "Add database connection unit test"
git push
````

## Step 5: Oracle Instant Client on Docker Container

We have [Oracle Instant Client](https://www.oracle.com/database/technologies/instant-client.html) installed on the development machine, but not on the build environment. And we need the ATP credential file to the instant client. Remember, our development environment is the Compute Instance on OCI with **Oracle Linux Server 7.7**, based on Cloud Developer Image. Our build, and future deployment environment, is a Docker image with **Debian GNU/Linux 10 (buster)** with Python 3, we get from Docker Hub, called **python:3.7**.

First, we need copy the ATP credential file to the git repository

```
$ mkdir wallet
$ cp ~/Downloads/Wallet_[Your Initials]ATP.zip ./wallet/
```

We have to add in **wercker.yml** two new steps to prepare our build and future deployment environment with Oracle Instant Client and prepare the credential file for ATP. The new steps has to be executed before the automated tests, make sure change the wallet file name to your own.

````
build:
    box: python:3.7
    steps:

    # Step 1: create virtual environment and install dependencies
    - script:
        name: install dependencies
        code: |
            python3 -m venv orclvenv
            . orclvenv/bin/activate
            pip install -r requirements.pip
    # Step 2: install Oracle client basic lite
    - script:
        name: install oracle-instantclient19.6
        code: |
            echo "deb http://ftp.debian.org/debian experimental main" >> /etc/apt/sources.list
            apt-get update
            apt-get install -y alien
            apt-get install -y libaio1
            wget https://download.oracle.com/otn_software/linux/instantclient/19600/oracle-instantclient19.6-basiclite-19.6.0.0.0-1.x86_64.rpm
            alien -i oracle-instantclient19.6-basiclite-19.6.0.0.0-1.x86_64.rpm
            export LD_LIBRARY_PATH=/usr/lib/oracle/19.6/client64/lib:$LD_LIBRARY_PATH
    # Step 3: unzip and copy ATP credential file
    - script:
        name: unzip and copy credential file
        code: |
            unzip -o -d /usr/lib/oracle/19.6/client64/lib/network/admin/ ./wallet/Wallet_[Your Initials]ATP.zip            
    # Step 4: run linter and tests
    - script:
        name: run tests
        code: |
            . orclvenv/bin/activate
            flake8 --exclude=orclvenv* --statistics
            pytest -v --cov=promotion
````

The step 2 adds a new package repository, installs two required packages, Oracle Instant Client 19.6, sets `LD_LIBRARY_PATH` environment variable. The step 3 unzip and copy the ATP credential file to the instant client directory.

Commit and push the changes.

````
git add wallet/Wallet_[Your Initials]ATP.zip
git commit -a -m "Add Oracle Instant Client"
git push
````

## Step 6: Web Publish Employees Table

Now we can add new features to our HR application. First feature will list all employees, whit their salary and commission. Add this new function with routing in promotion.py. 

````
@app.route('/employees')
def emp():
    sql = '''select FIRST_NAME, LAST_NAME, SALARY, COMMISSION_PCT,
                    SALARY * (1 + nvl(COMMISSION_PCT,0)) as "Total"
             from EMPLOYEES order by 1,2'''
    employees = '''<table border=1><tr><td>First Name</td><td>Last Name</td>
                   <td>Salary</td><td>Commission</td><td>Total</td></tr>'''
    cursor = connection.cursor()
    for res in cursor.execute(sql):
        employees += '<tr><td>' + res[0] + '</td><td>' + res[1] + '</td><td>' + str(res[2]) + '</td><td>' + str(res[3]) + '</td><td>' + str(res[4]) + '</td></tr>'
    employees += '</table>'
    return str(employees)
````

Test the web service locally, on the development environment.

````
python3 promotion.py 
Bottle v0.12.18 server starting up (using WSGIRefServer())...
Listening on http://0.0.0.0:8080/
Hit Ctrl-C to quit.
````

Use the web browser on your laptop to open [http://localhost:8080/employees](http://localhost:8080/employees). It shows a table with all employees in the HR schema. Hit Ctrl-C. Commit and push the changes.

````
git commit -a -m "Add new feature to list employees"
git push
````

Open Wercker console. Build pipeline returns and error:

````
./promotion.py:40:80: E501 line too long (162 > 79 characters)
````

By Python programming standards, all lines need to be under 80 characters. Our line number 40 has 162 characters. Change this new function with the following code, where that line is split in 3 separate lines:

````
@app.route('/employees')
def emp():
    sql = '''select FIRST_NAME, LAST_NAME, SALARY, COMMISSION_PCT,
                    SALARY * (1 + nvl(COMMISSION_PCT,0)) as "Total"
             from EMPLOYEES order by 1,2'''
    employees = '''<table border=1><tr><td>First Name</td><td>Last Name</td>
                   <td>Salary</td><td>Commission</td><td>Total</td></tr>'''
    cursor = connection.cursor()
    for res in cursor.execute(sql):
        employees += '<tr><td>' + res[0] + '</td><td>' + res[1] + '</td><td>'
        employees += str(res[2]) + '</td><td>' + str(res[3]) + '</td><td>'
        employees += str(res[4]) + '</td></tr>'
    employees += '</table>'
    return str(employees)
````

Commit and push the changes.

````
git commit -a -m "Fix long line problem"
git push
````

## Step 7: Add New Features to Web Service

Add one more feature that calculates an increase to all employees salary with a percentage.

````
@app.route('/salary_increase/<percentage>')
def sal_inc(percentage):
    sql = '''select FIRST_NAME, LAST_NAME, SALARY, COMMISSION_PCT,
                    SALARY * (1 + nvl(COMMISSION_PCT,0)) as "Total",
                    SALARY * (1 + ''' + percentage + '''/100) as "New Salary",
                    SALARY * (1 + ''' + percentage + '''/100) * (1 + nvl(COMMISSION_PCT,0)) as "New Total"
             from EMPLOYEES order by 1,2'''
    employees = '''<table border=1><tr><td>First Name</td><td>Last Name</td>
                   <td>Salary</td><td>Commission</td><td>Total</td>
                   <td>New Salary</td><td>New Total</td></tr>'''
    cursor = connection.cursor()
    for res in cursor.execute(sql):
        employees += '<tr><td>' + res[0] + '</td><td>' + res[1]
        employees += '</td><td>' + str(res[2]) + '</td><td>' + str(res[3])
        employees += '</td><td>' + str(res[4]) + '</td><td>' + str(res[5])
        employees += '</td><td>' + str(res[6]) + '</td></tr>'
    employees += '</table>'
    return str(employees)
````

And the final feature that adds a fixed values to the commission percentage to all employees.

````
@app.route('/add_commission/<value>')
def add_commp(value):
    sql = '''select FIRST_NAME, LAST_NAME, SALARY, COMMISSION_PCT,
                    SALARY * (1 + nvl(COMMISSION_PCT,0)) as "Total",
                    nvl(COMMISSION_PCT,0) + ''' + value + ''' as "New Commission",
                    SALARY * (1 + nvl(COMMISSION_PCT,0) + ''' + value + ''') as "New Total"
             from EMPLOYEES order by 1,2'''
    employees = '''<table border=1><tr><td>First Name</td><td>Last Name</td>
                   <td>Salary</td><td>Commission</td><td>Total</td>
                   <td>New Commission</td><td>New Total</td></tr>'''
    cursor = connection.cursor()
    for res in cursor.execute(sql):
        employees += '<tr><td>' + res[0] + '</td><td>' + res[1]
        employees += '</td><td>' + str(res[2]) + '</td><td>' + str(res[3])
        employees += '</td><td>' + str(res[4]) + '</td><td>' + str(res[5])
        employees += '</td><td>' + str(res[6]) + '</td></tr>'
    employees += '</table>'
    return str(employees)
````

Commit and push the changes.

````
git commit -a -m "Add final features"
git push
````

If all steps were followed correctly, this build is successful. We can test the application on the development environment now.

````
python3 promotion.py 
Bottle v0.12.18 server starting up (using WSGIRefServer())...
Listening on http://0.0.0.0:8080/
Hit Ctrl-C to quit.
````

In your browser open [`http://localhost:8080/salary_increase/8`](http://localhost:8080/salary_increase/8). It simulates a salary increase with 8% for all employees in our HR schema. Now open [`http://localhost:8080/add_commission/.15`](http://localhost:8080/add_commission/.15). This web service simulates adding 15% to the commission for all employees. 

Hit Ctrl-C to close the application.

In this lab, we were able to:

- Connect our Python microservice to Oracle Autonomous Database
- Create unit test for database connection per CI/CD requirements
- Install Oracle Instant Client on deployment environment
- Enhance our HR application with new features

## Acknowledgements

- **Author** - Valentin Leonard Tabacaru
- **Last Updated By/Date** - Minqiao Wang, DB Product Management, June 2020

See an issue? Please open up a request [here](https://github.com/oracle/learning-library/issues). Please include the workshop name and lab in your request.


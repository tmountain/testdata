# HOWTO Generate Test Data

This is a short guide demonstrating how to use synth to generate test data. 
The intended purpose is to generate mock data against an existing database schema.

### Install synth

```
sudo sh -c 'curl --proto '=https' --tlsv1.2 -sSL https://getsynth.com/install  | sh'
```

### Verify installation

```
% synth -h
synth 
synthetic data engine on the command line
...
```

### Install and start MySQL (assumes you have docker)

```
% cat stack.yml 

# Use root/example as user/password credentials
version: '3.1'

services:

    db:
	image: mysql
	command: --default-authentication-plugin=mysql_native_password
	restart: always
	environment:
	  MYSQL_ROOT_PASSWORD: root
	ports:
	  - 3306:3306

% docker-compose -f stack.yml up
```

### Install MySQL client using Nix

```
% nix-env -iA nixpkgs.mysql80
```

### Create a schema

```
% cat schema.sql 

CREATE DATABASE IF NOT EXISTS sandbox;
DROP TABLE IF EXISTS cow;
DROP TABLE IF EXISTS farmer;

CREATE TABLE `farmer` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `date` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `cow` (
  `id` int unsigned NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `farmer_id` int unsigned NOT NULL,
  `date` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (`id`),
  KEY `farmer_id` (`farmer_id`),
  CONSTRAINT `cow_ibfk_1` FOREIGN KEY (`farmer_id`) REFERENCES `farmer` (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

% mysql --protocol=tcp -u root -p -e 'CREATE DATABASE sandbox'
% mysql --protocol=tcp -u root -p sandbox < schema.sql
```

### Create data generators from the schema
```
% synth import sandbox --from mysql://root:root@localhost:3306/sandbox

% cat sandbox/farmer.json 
{
  "type": "array",
  "length": {
    "type": "number",
    "range": {
      "low": 0,
      "high": 2,
      "step": 1
    },
    "subtype": "u64"
  },
  "content": {
    "type": "object",
    "date": {
      "type": "date_time",
      "format": "",
      "subtype": "naive_date_time"
    },
    "id": {
      "type": "number",
      "id": {},
      "subtype": "i64"
    },
    "name": {
      "type": "string",
      "pattern": "[a-zA-Z0-9]{0, 255}"
    }
  }
}

% cat sandbox/cow.json
{
  "type": "array",
  "length": {
    "type": "number",
    "range": {
      "low": 0,
      "high": 2,
      "step": 1
    },
    "subtype": "u64"
  },
  "content": {
    "type": "object",
    "date": {
      "type": "date_time",
      "format": "",
      "subtype": "naive_date_time"
    },
    "farmer_id": {
      "type": "same_as",
      "ref": "farmer.content.id"
    },
    "id": {
      "type": "number",
      "id": {},
      "subtype": "i64"
    },
    "name": {
      "type": "string",
      "pattern": "[a-zA-Z0-9]{0, 255}"
    }
  }
}
```

### Tweak the generators to match your use case

The name fields will now use the first_name generator.

A maximum of 8 farmers will be generated with a maximum of 50 cows.

PK/FK ids will be mapped appropriately.

Timestamps will also be generated with range constraints if provided.

You can check these generators into your VCS and integrate them into your dev practice.

```
% cat sandbox/farmer.json 
{
  "type": "array",
  "length": {
    "type": "number",
    "range": {
      "low": 1,
      "high": 8,
      "step": 1
    },
    "subtype": "u64"
  },
  "content": {
    "type": "object",
    "date": {
      "type": "date_time",
      "format": "%Y-%m-%d %H:%M:%S",
      "subtype": "naive_date_time"
    },
    "id": {
      "type": "number",
      "id": {},
      "subtype": "i64"
    },
    "name": {
      "type": "string",
      "faker": {
          "generator": "first_name"
      }
    }
  }
}

% cat sandbox/cow.json
{
  "type": "array",
  "length": {
    "type": "number",
    "range": {
      "low": 1,
      "high": 50,
      "step": 1
    },
    "subtype": "u64"
  },
  "content": {
    "type": "object",
    "date": {
      "type": "date_time",
      "format": "%Y-%m-%d %H:%M:%S",
      "subtype": "naive_date_time",
      "begin": "2015-01-01 00:00:00",
      "end": "2020-01-01 12:00:00"
    },
    "farmer_id": {
      "type": "same_as",
      "ref": "farmer.content.id"
    },
    "id": {
      "type": "number",
      "id": {},
      "subtype": "i64"
    },
    "name": {
      "type": "string",
      "faker": {
          "generator": "first_name"
      }
    }
  }
}
```

### Install jq using Nix
```
% nix-env -iA nixpkgs.jq
```

### Generate data to confirm the format
```
% synth generate --size 50 sandbox | jq .

{
  "cow": [
    {
      "date": "2015-10-30 18:13:44",
      "farmer_id": 1,
      "id": 1,
      "name": "Diego"
    },
    {
      "date": "2019-12-05 00:52:54",
      "farmer_id": 2,
      "id": 2,
      "name": "Tatyana"
    },
    {
      "date": "2015-01-29 10:10:43",
      "farmer_id": 3,
      "id": 3,
      "name": "Fernando"
    },
    ...
  "farmer": [
    { 
      "date": "2021-12-09 16:19:05",
      "id": 1,
      "name": "Olin"
    },
    { 
      "date": "2021-12-09 16:19:05",
      "id": 2,
      "name": "Brody"
    },
    { 
      "date": "2021-12-09 16:19:05",
      "id": 3,
      "name": "Thora"
    },
    ...
}
```

### Write generated data to MySQL
```
% synth generate --size 50 sandbox --to mysql://root:root@localhost:3306/sandbox

% mysql -u root --protocol=tcp -p sandbox

mysql> select * from farmer limit 5;
+----+-------+---------------------+
| id | name  | date                |
+----+-------+---------------------+
|  1 | Olin  | 2021-12-09 16:26:07 |
|  2 | Brody | 2021-12-09 16:26:07 |
|  3 | Thora | 2021-12-09 16:26:07 |
|  4 | Conor | 2021-12-09 16:26:07 |
|  5 | Eli   | 2021-12-09 16:26:07 |
+----+-------+---------------------+
5 rows in set (0.00 sec)

mysql> select * from cow limit 5;
+----+----------+-----------+---------------------+
| id | name     | farmer_id | date                |
+----+----------+-----------+---------------------+
|  1 | Diego    |         1 | 2015-10-30 18:13:45 |
|  2 | Tatyana  |         2 | 2019-12-05 00:52:54 |
|  3 | Fernando |         3 | 2015-01-29 10:10:44 |
|  4 | Alva     |         4 | 2018-03-04 21:25:05 |
|  5 | Roman    |         5 | 2016-04-13 02:35:23 |
+----+----------+-----------+---------------------+
5 rows in set (0.01 sec)
```

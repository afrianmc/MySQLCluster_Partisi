# MySQLCluster_Partisi

Definisi :
Partisi adalah memecah tabel ke dalam beberapa segment (partisi atau subpartisi), di mana tabel konvensional hanya mempunyai satu segment.

*NB:*
Di dalam partisi tidak diperbolehkan menggunakan ```foreign key```, untuk mengatasi masalah tersebut adalah dengan mematikan atau comment 
foreign key
 didalam tabel tersebut
 
# Tipe Partisi
1. Range
Data dikelompokkan berdasarkan range(rentang) nilai yang kita tentukan. Range partition ini cocok digunakan pada kolom yang nilainya terdistribusi secara merata. Contoh yang paling sering adalah kolom tanggal.

2. List
Pada list partition, data dikelompokkan berdasarkan nilainya. Cocok untuk kolom yang variasi nilainya tidak banyak.

3. Hash
Jika kita ingin melakukan partisisi namun tidak cocok dengan RANGE ataupun LIST, maka kita bisa menggunakan HASH partition. Penentuan “nilai mana di taruh di partisi mana” itu diatur secara internal oleh Oracle (berdasarkan hash value).

4. Key
Partisi ini mirip dengan partisi HASH, perbedaannya hash menggunakan algoritma fungsi mysql standar yaitu MOD. sedangkan partisi key menggunakan algoritma yang memang didesain untuk data yang memiliki key jadi partisi ini digunakan untuk kolom yang memiliki key.

# Mengecek apakah plugin partition telah aktif
membuat database baru dengan nama ```test```.

```create database test;```

Memberikan izin agar user dapat mengakses database test melalui ProxySQL

```GRANT SELECT on test.* to 'user'@'%';```

```FLUSH PRIVILEGES;```

lalu mengecek Plugin partisi aktif 

```SHOW PLUGINS```
atau
```INFORMATION_SCHEMA.PLUGINS```

# Implementasi Partisi pada database Vertabelo
pada kasus ini kami menggunakan database vertabelo silahkan download terlebih dahulu [disini](https://drive.google.com/file/d/0B2Ksz9hP3LtXRUppZHdhT1pBaWM/view) 

## 1. Range Partition
```
CREATE TABLE lc (
    a INT NULL,
    b INT NULL
)
PARTITION BY LIST COLUMNS(a,b) (
    PARTITION p0 VALUES IN( (0,0), (NULL,NULL) ),
    PARTITION p1 VALUES IN( (0,1), (0,2), (0,3), (1,1), (1,2) ),
    PARTITION p2 VALUES IN( (1,0), (2,0), (2,1), (3,0), (3,1) ),
    PARTITION p3 VALUES IN( (1,3), (2,2), (2,3), (3,2), (3,3) )
);
```

## 2. Lish Partition
```
CREATE TABLE serverlogs (
    serverid INT NOT NULL, 
    logdata BLOB NOT NULL,
    created DATETIME NOT NULL
)
PARTITION BY LIST (serverid)(
    PARTITION server_east VALUES IN(1,43,65,12,56,73),
    PARTITION server_west VALUES IN(534,6422,196,956,22)
);
```

## 3. Hash Partision
```
CREATE TABLE serverlogs2 (
    serverid INT NOT NULL, 
    logdata BLOB NOT NULL,
    created DATETIME NOT NULL
)
PARTITION BY HASH (serverid)
PARTITIONS 10;
```

## 4. Key Partition
```
CREATE TABLE serverlogs4 (
    serverid INT NOT NULL, 
    logdata BLOB NOT NULL,
    created DATETIME NOT NULL,
    UNIQUE KEY (serverid)
)
PARTITION BY KEY()
PARTITIONS 10;
```

# Testing pada Range Partition
### Gunakan perintah EXPLAIN untuk melihat plan eksekusi query untuk masing-masing tabel.
> EXPLAIN tanpa Partisi
```
EXPLAIN SELECT *
FROM test.measures
WHERE measure_timestamp >= '2016-01-01' AND DAYOFWEEK(measure_timestamp) = 1;
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/1.png)

> EXPLAIN dengan Partisi
```
EXPLAIN PARTITIONS SELECT *
FROM test.partitioned_measures
WHERE measure_timestamp >= '2016-01-01' AND DAYOFWEEK(measure_timestamp) = 1;
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/2.png)

### Jalankan query benchmark untuk masing-masing tabel. Hasilnya adalah running time.
> Select Benchmark tanpa Partisi
```
SELECT SQL_NO_CACHE
    COUNT(*)
FROM
    vertabelo.measures
WHERE
    measure_timestamp >= '2016-01-01'
        AND DAYOFWEEK(measure_timestamp) = 1;
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/3.png)

> Select Benchmark dengan partisi
``` 
SELECT SQL_NO_CACHE
    COUNT(*)
FROM
    vertabelo.partitioned_measures
WHERE
    measure_timestamp >= '2016-01-01'
        AND DAYOFWEEK(measure_timestamp) = 1;
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/4.png)

### Jalankan query delete (bagian BIG DELETE) dan tampilkan perbedaan running time-nya.
> Menambah data tanpa Partisi
```
ALTER TABLE `test`.`measures` 
ADD INDEX `index1` (`measure_timestamp` ASC);
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/5.png)

> Menghapus data tanpa Partisi
```
ALTER TABLE `test`.`measures` 
DROP INDEX `measure_timestamp` ;
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/6.png)

> Menambah data dengan Partisi
```
ALTER TABLE `test`.`partitioned_measures` 
ADD INDEX `index1` (`measure_timestamp` ASC);
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/7.png)

> Menghapus data dengan Partisi
```
ALTER TABLE `test`.`partitioned_measures` 
DROP INDEX `measure_timestamp` ;
```
![Ss](https://github.com/afrianmc/MySQLCluster_Partisi/blob/master/screenshot/8.png)

# Referensi
https://www.vertabelo.com/blog/technical-articles/everything-you-need-to-know-about-mysql-partitions

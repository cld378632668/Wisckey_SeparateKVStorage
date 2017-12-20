# ycsb-leveldb-leveldbjni-rocksdb
ycsb-leveldb-leveldbjni-rocksdb


#leveldb-binding


use net connection, localhost:8080read

#leveldbjni-binding
level


# RocksDB YCSB Binding
YCSB(Yahoo! Cloud System Benchmark) binding to RocksDB.

## To compile
javac -cp  \<directory-to-rocksdbjni-jar\>/rocksdbjni.jar:\<directory-to-ycsb-core-jar\>/core-0.1.4.jar \<directory-to-java-source-root\>/**.java

## To load
java -ea -cp \<directory-to-compiled-classes\>:\<directory-to-rocksdbjni-jar\>/rocksdbjni.jar:\<directory-to-ycsb-core-jar\>/core-0.1.4.jar com.yahoo.ycsb.Client -db org.rocksdb.ycsb.RocksDBYcsbBinding -P \<workload-file\> -load

## To run
java -ea -cp \<directory-to-compiled-classes\>:\<directory-to-rocksdbjni-jar\>/rocksdbjni.jar:\<directory-to-ycsb-core-jar\>/core-0.1.4.jar com.yahoo.ycsb.Client -db org.rocksdb.ycsb.RocksDBYcsbBinding -P \<workload-file\> -t

@echo off
TITLE MongoDB Portable Server
:: Path to your bin folder
SET MONGO_BIN="C:\Users\Atul\MongoDB\bin\mongod.exe"
:: Path to your data folder
SET DB_DATA="C:\Users\Atul\MongoDB\data"
:: Path to your log folder
SET DB_LOG="C:\Users\Atul\MongoDB\logs\mongod.log"

echo Starting MongoDB...
%MONGO_BIN% --dbpath %DB_DATA% --logpath %DB_LOG% --logappend
pause

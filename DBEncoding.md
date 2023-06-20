Script to update encoding for MySQL databases with mix of UTF-8 and Latin1 encoding:

```
$db = \DB::connect();
$sqlDbs = 'SHOW DATABASES';
$dbs = $db->selectArray($sqlDbs);
foreach ($dbs as $database) {
    $dbName = $database['Database'];
    if (! in_array($dbName, ['information_schema', 'sys', 'mysql', 'performance_schema'])) {
        $sqlTables = "SHOW TABLES FROM {$dbName}";
        $tables = $db->selectArray($sqlTables);
        foreach ($tables as $table) {
            $tableParams = ['db' => $dbName, 'table' => $table[0]];
            $fullTableName = "{$dbName}.`{$table[0]}`";

            $charsetSql = 'SELECT CCSA.character_set_name AS charset FROM information_schema.tables T,
                        information_schema.collation_character_set_applicability CCSA
                WHERE CCSA.collation_name = T.table_collation
                AND T.table_schema = :db
                AND T.table_name = :table';
            $charset = array_values($db->selectArray($charsetSql, $tableParams));
            $charset = $charset[0]['charset'];

            $sqlTableDesc = "DESC {$fullTableName}";
            $tableDesc = $db->selectArray($sqlTableDesc);

            $colEncodingData = getColumnEncodingDataForTable($db, $tableParams);
            $colConstraints = getForeignConstraintsForTable($db, $tableParams);
            $blockingIndices = getBlockingIndicesForTable($db, $tableParams);

            list($hasTextCols, $sql, $blockingSql) = convertTableCommands($fullTableName, $tableDesc, $charset, $colEncodingData, $colConstraints, $blockingIndices);

            // Write commands to file
            
            if (! empty($colConstraints)) {
                $foreignConstraintContent = formatForeignConstraints($fullTableName, $colConstraints);
                file_put_contents('foreignConstraints.txt', $foreignConstraintContent, FILE_APPEND | LOCK_EX);
            }
            if (! empty($blockingIndices)) {
                // $blockingIndicesContent = formatBlockingIndices($fullTableName, $blockingIndices);
                file_put_contents('blockingIndices.txt', $blockingSql, FILE_APPEND | LOCK_EX);
            }
        }
    }
}

function convertTableCommands($fullname, $tableDesc, $charset, $colEncodingData, $colConstraints, $fullTextIndices)
{
    $sql = "\nALTER TABLE {$fullname} CHARACTER SET utf8 COLLATE utf8_unicode_ci;\n";
    $blockingIdxSql = '';
    $foundTextCol = false;
    foreach ($tableDesc as $col) {
        $type = $col['Type'];
        $colName = $col['Field'];

        if (key_exists($colName, $colConstraints)) {
            continue;
        }

        // Set variables based on column data type

        if ($isAnyText) {
            $migrationSql = '';
            $isBlockingIndex = key_exists($colName, $fullTextIndices);
            if ($isBlockingIndex) {
                $blockingIdxSql .= "\n\n";
                $blockingIdxSql .= $fullTextIndices[$colName]['drop'];
            }

            $colCharset = \Util::fetch($colEncodingData, [$colName, 'character_set_name'], $charset);
            if ($colCharset == 'latin1') {
                $migrationSql .= "ALTER TABLE {$fullname} MODIFY COLUMN `{$colName}` {$binType}{$typeLength} {$allowsNull} {$default};\n";
            } else {
                $migrationSql .= "UPDATE {$fullname} SET `{$colName}` = CONVERT(CAST(CONVERT(`{$colName}` USING latin1) AS binary) USING utf8) WHERE CONVERT(CAST(CONVERT(`{$colName}` USING latin1) AS binary) USING utf8) IS NOT NULL;\n";
                $colCollation = \Util::fetch($colEncodingData, [$colName, 'collation_name']);
                $requireAlter = ($colCollation != 'utf8_unicode_ci');
            }

            if ($requireAlter) {
                $migrationSql .= "ALTER TABLE {$fullname} MODIFY COLUMN `{$colName}` {$type} CHARACTER SET utf8 COLLATE utf8_unicode_ci {$allowsNull} {$default};\n";
            }

            if ($trimPadding && $colCharset == 'latin1') {
                $migrationSql .= "UPDATE {$fullname} SET `{$colName}` = TRIM(TRAILING UNHEX('0') FROM `{$colName}`);\n";
            }

            if ($isBlockingIndex) {
                $blockingIdxSql .= $migrationSql;
                $blockingIdxSql .= $fullTextIndices[$colName]['add'];
            } else {
                $sql .= $migrationSql;
            }
        }
    }

    return [$foundTextCol, $sql, $blockingIdxSql];
}

function formatForeignConstraints($tableName, $constraints) { }

function formatBlockingIndices($tableName, $indices) { }

function getBlockingIndicesForTable($db, $tableParams) { }

function getForeignConstraintsForTable($db, $tableParams) { }

function getColumnEncodingDataForTable($db, $tableParams) { }
```

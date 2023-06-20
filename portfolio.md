Initial implementation was a single ~800 line `scan:` function.

```
- (id)init;
- (void)scan:(UIImage *)image;
```

Refactored to add 11 helper functions (~45 lines each) with names to describe function of underlying code while also fixing reliability of scanning from ~80% to ~99%:

```
- (id)init;

- (void)setHorizontalSpacingFromTotalWidth:(float)totalWidth;

- (void)setVerticalSpacingFromTotalHeight:(float)totalHeight;

-(UIImage *)UIImageFromCVMatRef:(cv::Mat *)cvMatRef;

- (BOOL)imageIsBlack:(cv::Mat)image atCol:(int)col andRow:(int)row;

- (void)findRightColumnEdges:(cv::Mat)binary rightY:(int*)righty leftY:(int*)lefty leftX:(int*)leftx rightX:(int*)rightx;

- (void)findLeftColumnEdges:(cv::Mat)binary rightY:(int*)rightY leftY:(int*)leftY rightX:(int*)rightX leftX:(int*)leftX;

- (void)gradeLeftColumnQuestionsForImage:(cv::Mat)binary rightY:(int)rightY leftY:(int)leftY rightX:(int)rightX leftX:(int)leftX answerImage:(cv::Mat)binary2;

- (void)gradeRightColumnQuestion:(cv::Mat)binary rightY:(int)rightY leftY:(int)leftY rightX:(int)rightX leftX:(int)leftX answerImage:(cv::Mat)binary2 questionNum:(int)numq;

- (void)findLanguageAndForm:(cv::Mat)binary rightY:(int)rightY leftY:(int)leftY rightX:(int)rightX leftX:(int)leftX answerImage:(cv::Mat)binary2;

- (void)parseCfidArea:(cv::Mat)binary answerImage:(cv::Mat)binary2 bottomRightX:(int)bottomRightX rowPositionsY:(NSMutableArray *)rowPositionsY;

- (cv::Mat)setThresholdFromGreyscale:(cv::Mat)greyMat image:(cv::Mat)mat;

- (void)scan:(UIImage *)image;
```

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

Adding support for coupons to an existing Invoices API that supports payments with Stripe.

```
<?php

namespace Invoices\Helpers;

// Use statements deleted

trait InvoicesCouponTrait
{
    /*
     * A “dry run” means that the coupon is applied in a pending
     * state. Think of a customer adding a coupon code in a 
     * checkout screen but the coupon hasn’t been used until the
     * transaction is completed.
     */
    public function applyCoupon($coupon, $dryRun = false)
    {
        // Validity checks deleted

        if ($coupon->isVoucher) {
            return $this->applyVoucherCoupon($coupon, $dryRun);
        }

        return $this->applyDiscountCoupon($coupon, $dryRun);
    }

    /*
     * A “voucher” coupon is a coupon that is used like a gift
     * certificate and is applied as a form of payment.
     */
    public function applyVoucherCoupon($coupon, $dryRun)
    {
       if ($dryRun) {
            // For a dry run, add a pending status with the
            //   coupon details
            $this->updateStatus(Invoice::STATUS_PENDING, $coupon->identifier, $transactionDetails);
      } else {
            // Look for the pending status and update it
            foreach ($this->getActiveStatuses() as $status) {
                if ($status->transactionId == $coupon->identifier) {
                    $status->status = Invoice::STATUS_PAID;
                    $status->saveAndNotify();

                    $coupon->markAsUsed();
                    // Inform other systems about coupon redemption
               }
            }
        }
       return $updatedInvoiceData;
    }

    /*
     * A discount coupon changes the price to be paid. This
     * has tax implications so it must be applied before taxes
     * are applied. It is modelled as a negative line item.
     */
    public function applyDiscountCoupon($coupon, $dryRun)
    {
        $couponAmount = -$coupon->amount;

       if ($dryRun) {
            list($ok, $details) = $this->addPendingCoupon($coupon);
            if ($ok) {
                $this->updateTax();
                return $updatedInvoiceData;
            }
           return $errorData;
        }

        $item = $this->lineItemFromCoupon($coupon);

        // If this coupon was pending, it isn't anymore
        $this->removePendingCoupon($coupon);

        $this->updateTax();
        $this->save();

        $coupon->markAsUsed();
        // Inform other systems about coupon redemption

        return $updatedInvoiceData;
    }

    /*
     * A common scenario is a customer registers for a seminar,
     * then realizes they’re eligible for a discount. We need
     * to retroactively apply the coupon and issue a refund for
     * the difference.
     */
    public function issueRefundApplyCoupon($code)
    {
        $coupon = Coupons::couponByEntityAndCode($this->entityType, $this->entityId, $code);

        // Perform error checking

        $initialDue = $this->calculateAmountDue();
        $couponResult = $this->applyCoupon($coupon, false, $userid);

        $updatedDue = $this->calculateAmountDue();

        // Now we need to issue a refund for the difference
        $refundAmount = $initialDue - $updatedDue;
        if ($refundAmount > 0) {
            $statuses = $this->getActiveStatuses();
            foreach ($statuses as $s) {
                // Refunds need to be issued against individual
                //   payments in Stripe.
                if ($refundAmount > 0 && $s->status == Invoice::STATUS_PAID) {
                    $statusRefundAmount = min($refundAmount, $s->amountPaid(false));
                    StripeRefund::issueRefund($this, $statusRefundAmount);
                    $refundAmount -= $statusRefundAmount;
               }
            }
        }
        return $couponResult;
    }

    public function lineItemFromCoupon($coupon)
    {
        $couponAmount = -$coupon->amount;
        $lineItemData = [
          // Line item properties
        ];

        $item = new LineItems();
        $item->invoice = $this;
        $item->assign($lineItemData);
        return $item;
    }

    /*
     * These functions are more involved with the inner workings
     * of invoice objects.
     */

    public function pendingPayments() { }

    public function applyPendingCoupons() { }

    public function pendingCouponAmount() { }

    public function pendingPaymentAmount() { }

    public function addPendingCoupon($coupon) { }

    public function removePendingCoupon($coupon) { }
```

Implementing PayPal support through CyberSource: https://certifications.crossfit.com/checkout/invoices/61ae6153869af

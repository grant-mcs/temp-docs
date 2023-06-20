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

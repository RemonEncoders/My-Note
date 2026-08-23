| Controller Name                  | When to Use                                                             |
| -------------------------------- | ----------------------------------------------------------------------- |
| `StockAdjustmentController`      | If all transactions are inventory adjustments (increase/decrease).      |
| `InventoryAdjustmentController`  | More professional and broader term for stock modifications.             |
| `StockIssueController`           | If items are issued out from inventory for any reason.                  |
| `InventoryIssueController`       | Similar to StockIssue but more enterprise-friendly.                     |
| `StockDispositionController`     | If the purpose is disposal, damage, return, consumption, etc.           |
| `InventoryTransactionController` | If it may also include stock-in transactions in the future.             |
| `StockOutController`             | Acceptable if the controller only handles outbound inventory movements. |

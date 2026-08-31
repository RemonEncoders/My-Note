## দুটো জায়গায় ইচ্ছাকৃতভাবে সংযত থেকেছি

**পুরনো PO সম্পাদনাযোগ্য থাকে।** কেউ নিষিদ্ধ হওয়ার আগে দেওয়া অর্ডারের supplier তালিকায় ফিরিয়ে আনা হয় ([RestoreBlacklistedSupplierAsync](vscode-webview://0tri44ar51et483btke71i178mqksugck1rce8n1q6b1u371eioe/HMS.Web/Areas/SCM/Models/PurchaseOrderModel.cs#L318)), আর server-side যাচাই কেবল **supplier বদলালে** চালু হয়। নইলে ব্যানারের পর ওই PO-র অন্য কিছু ঠিক করতে গেলেই supplier নীরবে উবে যেত — ঠিক যে শ্রেণির বাগ থেকে আমরা পালাচ্ছি।

**কেন্দ্রীয় `GetVendorListAsync`-এ filter দিইনি** — payment স্ক্রিন ওটাই ব্যবহার করে; blacklisted vendor-এর পুরনো বকেয়া তো শোধ করতেই হবে।
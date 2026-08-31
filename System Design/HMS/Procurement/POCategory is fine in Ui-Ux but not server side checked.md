# ১. "Category-purity gap"-টা আসলে কোন gap?

সোজা উত্তর: **এটা আগের ৫টা gap-এর কোনোটাই নয় — এটা একটা নতুন, ষষ্ঠ ফাঁক**, যেটা গতকাল আপনার requirement ১ বলার পরেই ধরা পড়ল। আগের ৫টা ছিল status/write-back নিয়ে; এটা হলো **PO-র বিষয়বস্তু নিয়ে**।

কোডে এখন ঠিক কী অবস্থা:

|জিনিস|আছে কি?|
|---|---|
|PO-তে category ধারণা (`POCategoryId` — Goods/Service parent)|✅ আছে|
|PO Add screen-এ PR-এর item গুলো category অনুযায়ী filter|✅ আছে, কিন্তু **শুধু client-side (JavaScript-এ)**|
|Server-এ যাচাই যে PO-র সব line তার category-র সাথে মেলে|❌ **নেই — এটাই gap**|

মানে: এখন একজন user সাবধানে UI ব্যবহার করলে ঠিকই category-pure PO হয়, কিন্তু **সিস্টেম কোথাও আটকায় না** — ভুল করে বা hand-crafted POST দিয়ে এক PO-তে goods+service মিশে গেলে DB তা মেনে নেয়। আপনার ব্যবসায়িক নিয়ম ("এক PO-তে এক category") এখন **নিয়ম হিসেবে কোথাও লেখা নেই, শুধু অভ্যাস হিসেবে আছে।**

এটাকে "Scenario 2-এর gap" বললাম কারণ এটা PR→PO ধাপের সমস্যা — **RFQ থাকুক বা না থাকুক, এই ধাপ দুই scenario-তেই চলে।** তাই এটা RFQ-র আগে আলাদাভাবে সারা যায়।
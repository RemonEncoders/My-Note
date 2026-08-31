আমার আরেকটি UI/UX সম্পর্কিত Business Requirement আছে।

বর্তমানে **Create PO from RFQ**-এর Item Table এবং **Direct PO Create**-এর Item Table-এর UI ও Layout এক নয়। আমার মতে, দুটি Item Table-এর Structure, Column Layout এবং User Experience একই হওয়া উচিত।

কারণ ব্যবহারকারীর দৃষ্টিকোণ থেকে—উভয় ক্ষেত্রেই তিনি একটি Purchase Order-ই তৈরি করছেন। শুধু Data Source আলাদা (একটি Direct PO, অন্যটি RFQ থেকে)। তাই UI ভিন্ন হওয়ার কোনো Business Value নেই; বরং এটি ব্যবহারকারীকে বিভ্রান্ত করতে পারে এবং প্রতিবার নতুনভাবে UI বুঝতে বাধ্য করে।

আমার প্রস্তাব হলো:

- **Create PO from RFQ**-এর Item Table-কে **Direct PO Create**-এর Item Table-এর মতোই করা হোক।
    
- একই Column Order, একই Layout, একই Input Controls এবং একই User Interaction বজায় রাখা হোক।
    
- শুধুমাত্র যেসব Field RFQ থেকে স্বয়ংক্রিয়ভাবে আসে (যেমন Winner Vendor, Winner Unit Price ইত্যাদি), সেগুলোর Data Source আলাদা হবে। কিন্তু UI ও ব্যবহার পদ্ধতি একই থাকবে।
    

এতে ব্যবহারকারী একটি Consistent Experience পাবে, Training-এর প্রয়োজন কমবে এবং UI Switching-এর কারণে যে Cognitive Load তৈরি হয় তা অনেকটাই কমে যাবে। একই ধরনের কাজের জন্য একই ধরনের Interface ব্যবহার করাই Enterprise ERP System-এর একটি ভালো UI/UX Practice।
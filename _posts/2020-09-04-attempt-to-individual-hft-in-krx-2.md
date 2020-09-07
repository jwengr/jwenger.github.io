---
published: true
---
In this page, I'll explain my HFT trading flow.  
The flow likes this.
![flow](/assets/img/flow.png)
It is not complecated and not perfect. I plan to supplement this by implementing it.

### 1.Get xml data from KRX  
KRX(Korea Exchange) provides real-time stock price information through XML data in web source page. Using c++ library, data will be collected and parsed.

### 2.Save data in Shared Memory and share with Python
Trading logic will be put into practice through Python. In order to do, those processes share data through Shared Memory related libraries(shm in c++, sysv_ipc in python). 

### 3.Calculate data and make Ask/Bid.
Compute that data through Python and send the result(Ask/Bid signal) to the Windows. Windows works with stock firm's API to make actual Ask/Bid.

### liquidity strategy
Maybe it should be done in Windows.


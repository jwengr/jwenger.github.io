---
published: true
---
### Get xml -> parsing(c++) -> shared memory -> parsing(python) -> dataframe
#### source page
![xmlexample](/assets/img/xmlexample.png)
  
#### result
![shmbytes](/assets/img/shmbytes.png)
![result](/assets/img/result.png)  
  
### Full Code
~~~c++
#include <iostream>
#include <curl/curl.h>
#include <string>
#include <sys/shm.h>
#include <sys/ipc.h>
#include "./rapidxml-1.13/rapidxml.hpp"
#include <cstring>

static size_t WriteCallBack(void *contents, size_t size, size_t nmemb, void *userp){
    ((std::string*)userp)->append((char*)contents, size * nmemb);
    return size * nmemb;
}

std::string GetXml(std::string url){
        CURL *curl;
        CURLcode res;
        std::string readBuffer;
        curl = curl_easy_init();
        if(curl) {
                curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
                curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallBack);
                curl_easy_setopt(curl, CURLOPT_WRITEDATA, &readBuffer);
                res = curl_easy_perform(curl);
        }
        curl_easy_cleanup(curl);
        curl_global_cleanup();

        return readBuffer;
}

std::string StrToDict(std::string s1, std::string s2){
        std::string result =  (std::string)"\'" + s1 + (std::string)"\':\'" + s2 +"\',";
        return result;
}


std::string XmlPars(std::string xml,std::string code){
        std::string result = "{";
        result += StrToDict("code",code);
        rapidxml::xml_document<> doc;
        rapidxml::xml_node<> *root_node;
        doc.parse<0>(&xml[0]);
        root_node = doc.first_node("stockprice");
        result += StrToDict("time",root_node->first_attribute("querytime")->value());
        rapidxml::xml_node<> *ta_node = root_node->first_node("TBL_AskPrice");
        int i = 0;
        for(rapidxml::xml_node<> *ask_node = ta_node->first_node("AskPrice"); ask_node ; ask_node = ask_node->next_sibling()){
                result += StrToDict("BidTop"+std::to_string(++i),ask_node->first_attribute("member_memdoVol")->value());
                result += StrToDict("AskTop"+std::to_string(i),ask_node->first_attribute("member_mesuoVol")->value());
        }
        rapidxml::xml_node<> *ts_node = root_node->first_node("TBL_StockInfo");
        result += StrToDict("Cur",ts_node->first_attribute("CurJuka")->value());
        result += StrToDict("Vol",ts_node->first_attribute("Volume")->value());
        rapidxml::xml_node<> *th_node = root_node->first_node("TBL_Hoga");
        for(i=0;i<5;i++){
                result += StrToDict("BookAskPrice"+std::to_string(i),th_node->first_attribute((const char*)("mesuHoka"+std::to_string(i)).c_str())->value());
                result += StrToDict("BookAskVol"+std::to_string(i),th_node->first_attribute((const char*)("mesuJan"+std::to_string(i)).c_str())->value());
                result += StrToDict("BookBidPrice"+std::to_string(i),th_node->first_attribute((const char*)("medoHoka"+std::to_string(i)).c_str())->value());
                result += StrToDict("BookBidVol"+std::to_string(i),th_node->first_attribute((const char*)("medoJan"+std::to_string(i)).c_str())->value());
        }
        result.pop_back();
        result += "}";
        return result;
}


int SHM(std::string str,std::string code){
        int shm_id;
        void *shm_addr;
        if(-1 == (shm_id = shmget((key_t)std::stoi(code),1024,IPC_CREAT|0666))){
                std::cout << "shmget failed" << std::endl;
                return -1;
        }
        if((void*)-1 == (shm_addr = shmat(shm_id,NULL,0))){
                std::cout << "shmat failed" << std::endl;
                return -1;
        }
        memcpy(shm_addr,str.c_str(),str.size());
        return 0;
}

int main(void){
        std::string code = "035420";
        std::string prefix = "http://asp1.krx.co.kr/servlet/krx.asp.XMLSise?code=";
        std::string tempurl = "";
        std::string url = prefix+code;
        std::string xml =  GetXml(url);
        std::string par = XmlPars(xml,code);
        SHM(par,code);
        return 0;
}  
~~~
### 1.Get Xml by using curlib
~~~c++
#include <iostream>
#include <curl/curl.h>
#include <string>

std::string GetXml(std::string url){
        CURL *curl;
        CURLcode res;
        std::string readBuffer;
        curl = curl_easy_init();
        if(curl) {
                curl_easy_setopt(curl, CURLOPT_URL, url.c_str());
                curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, WriteCallBack);
                curl_easy_setopt(curl, CURLOPT_WRITEDATA, &readBuffer);
                res = curl_easy_perform(curl);
        }
        curl_easy_cleanup(curl);
        curl_global_cleanup();

        return readBuffer;
}
~~~
### 2.Parsing Xml by RapidXml
~~~c++
#include "./rapidxml-1.13/rapidxml.hpp"
#include <cstring>

std::string StrToDict(std::string s1, std::string s2){
        std::string result =  (std::string)"\'" + s1 + (std::string)"\':\'" + s2 +"\',";
        return result;
}


std::string XmlPars(std::string xml,std::string code){
        std::string result = "{";
        result += StrToDict("code",code);
        rapidxml::xml_document<> doc;
        rapidxml::xml_node<> *root_node;
        doc.parse<0>(&xml[0]);
        root_node = doc.first_node("stockprice");
        result += StrToDict("time",root_node->first_attribute("querytime")->value());
        rapidxml::xml_node<> *ta_node = root_node->first_node("TBL_AskPrice");
        int i = 0;
        for(rapidxml::xml_node<> *ask_node = ta_node->first_node("AskPrice"); ask_node ; ask_node = ask_node->next_sibling()){
                result += StrToDict("BidTop"+std::to_string(++i),ask_node->first_attribute("member_memdoVol")->value());
                result += StrToDict("AskTop"+std::to_string(i),ask_node->first_attribute("member_mesuoVol")->value());
        }
        rapidxml::xml_node<> *ts_node = root_node->first_node("TBL_StockInfo");
        result += StrToDict("Cur",ts_node->first_attribute("CurJuka")->value());
        result += StrToDict("Vol",ts_node->first_attribute("Volume")->value());
        rapidxml::xml_node<> *th_node = root_node->first_node("TBL_Hoga");
        for(i=0;i<5;i++){
                result += StrToDict("BookAskPrice"+std::to_string(i),th_node->first_attribute((const char*)("mesuHoka"+std::to_string(i)).c_str())->value());
                result += StrToDict("BookAskVol"+std::to_string(i),th_node->first_attribute((const char*)("mesuJan"+std::to_string(i)).c_str())->value());
                result += StrToDict("BookBidPrice"+std::to_string(i),th_node->first_attribute((const char*)("medoHoka"+std::to_string(i)).c_str())->value());
                result += StrToDict("BookBidVol"+std::to_string(i),th_node->first_attribute((const char*)("medoJan"+std::to_string(i)).c_str())->value());
        }
        result.pop_back();
        result += "}";
        return result;
}
~~~
### 3.Copy result to Shared Memory
~~~c++
#include <sys/shm.h>
#include <sys/ipc.h>

int SHM(std::string str,std::string code){
        int shm_id;
        void *shm_addr;
        if(-1 == (shm_id = shmget((key_t)std::stoi(code),1024,IPC_CREAT|0666))){
                std::cout << "shmget failed" << std::endl;
                return -1;
        }
        if((void*)-1 == (shm_addr = shmat(shm_id,NULL,0))){
                std::cout << "shmat failed" << std::endl;
                return -1;
        }
        memcpy(shm_addr,str.c_str(),str.size());
        return 0;
}

int main(void){
        std::string code = "035420";
        std::string prefix = "http://asp1.krx.co.kr/servlet/krx.asp.XMLSise?code=";
        std::string tempurl = "";
        std::string url = prefix+code;
        std::string xml =  GetXml(url);
        std::string par = XmlPars(xml,code);
        SHM(par,code);
        return 0;
}  
~~~

### 4. Read Shared Memory and Preprocessing with Python3
~~~python
import sysv_ipc as shm
from ast import literal_eval
import pandas as pd

df = None
code = "035420"
shmString = shm.SharedMemory(int(code))
shmString.read()
string = shmString.read().decode("utf-8")
shmString.remove()
string = string[:string.find("}")+1]
dic = literal_eval(string)
if df is None : 
    df = pd.DataFrame.from_dict([dic]).set_index('time')
else : 
    temp = pd.DataFrame.from_dict([dic]).set_index('time')
    df = pd.concat([df,temp],axis=0)
~~~
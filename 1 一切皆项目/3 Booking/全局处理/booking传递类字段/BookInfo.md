# BookInfo

该字段主要用于进订阶段的预定信息向下单传递。

**写入**

该字段的写入阶段在进订的结果处理Item里面，会有一个`BookInfoBookingAdaptor.fillBookInfo(parameter);`​来进行处理。塞入的内容大概是：酒店房型、支付要求、价格日历、入理日期、取消规则、extraParam等信息。

> extraParam为啥要在bookinfo再放一份？

**读取**

* 下单的时候取预订所用的酒店信息、入离日期等；
* 发送变价消息时需要用到酒店的一些信息；
* 是否需要英文入住人姓名，需要在此取控制字段；

  > 为啥不用下单的报价信息？
  >

* 取其中的extraParam字段。
* 略

**case**

```json
{
    "score": "",
    "dangci": "1",
    "from": 0,
    "fromDate": "2024-09-12",
    "toDate": "2024-09-13",
    "otaPhone": "01089954155转03604",
    "wrapperId": "hta10c5vo3a",
    "hotelSeq": "beijing_city_126423",
    "hotelId": "485710622",
    "roomId": "1394489484_1588566",
    "hotelPhone": "+861062612991",
    "hotelAddress": "北京海淀区万泉河路58号",
    "planId": "",
    "datePrice": [
        {
            "day": "2024-09-12",
            "price": "566.00",
            "salePrice": "566.00",
            "status": 0,
            "cut": 0,
            "remain": "8",
            "bookable": true
        }
    ],
    "cityName": "北京",
    "currencyCode": "CNY",
    "taxationCurrency": "CNY",
    "ptType": 0,
    "extraParam": "{...}",
    "referenceCurrency": "CNY",
    "countryCode": "",
    "redEnvelopeMoney": 0,
    "priceFrom": 0,
    "mobilePriceType": 38,
    "inventoryType": 0,
    "specialProp": "{}",
    "checkInCashBack": 0.0,
    "creditUserType": 1,
    "allowBindCardLater": true,
    "logicBit": 0,
    "breakfast": "无早",
    "payType": 0,
    "webFree": "",
    "wifi": "",
    "fromForLog": 913,
    "guestNameType": 1,
    "needEnglishGuestName": false,
    "cancelType": "CANCEL_TIME_LIMIT",
    "freeCancelTime": "2024-09-12 18:00:00",
    "payCancelTime": "",
    "bindCardType": 0,
    "instantCount": 8,
    "quitTipsSortId": {
        "A": 1,
        "B": 2,
        "C": 3,
        "D": 4,
        "E": 5,
        "F": 6,
        "G": 7
    },
    "bookingRenderingTipsNum": 2,
    "groupDiscounted": 0,
    "saleUnit": "ROOM",
    "datePriceItems": [
        {
            "day": "2024-09-12",
            "price": "566.00",
            "text": "{bookNum}间"
        }
    ],
    "flashLodgingVer": 0,
    "ifContainCashBack": false,
    "perfectPlan": false,
    "filterPackageTypes": "",
    "priceInfos": [
        {
            "bookNum": 1,
            "totalPrice": "566",
            "totalPrize": "8",
            "referTotalPrice": "",
            "prepayAmount": "",
            "overagePrice": "",
            "roomCommissionPrice": "187.46",
            "roomOriginTotalPrice": "",
            "selected": true,
            "totalVouchMoney": ""
        }...
    
    ],
    "actualRoomOccupancy": 0,
    "sId": "PSS|245712543|beijing_city_126423|2024-09-12|2024-09-13|4925673677569922|675|zstd",
    "supportVcc": false,
    "roomType": "2",
    "bookingNewFlow": true,
    "rewardPoint": 5813,
    "maxAdvancePoint": 30000,
    "rewardTimes": {
        "9058": 0,
        "9056": 2,
        "9057": 1,
        "9050": 2,
        "9061": 100,
        "9051": 2,
        "9062": 100,
        "9054": 0,
        "9052": 2,
        "9053": 2
    },
    "travelMaskInfo": {
        "residenceMask": {
            "provinceEqual": true,
            "dijiEqual": true,
            "countyEqual": false,
            "cityId": "1",
            "cityUrl": "beijing_city"
        }
    },
    "integralBlackUser": false,
    "userBizIdentity": "[\"ncsu\",\"entitlement\",\"allc\"]",
    "saleGroupChannel": "MS",
    "promotionFlag": 0,
    "useMarketCoin": false
}
```

## `refreshExtra`​

‍

‍

---
layout: post
title:  "Generic"
categories: geco
tags:  python
---

* content
{:toc}


## Forms
**_forms** is used to indicate what request formate to be accepted by server in next game. You can find it in response section in debug page and it is something that looks like this:
```json
"_forms": {
    "application/vnd.miko.slots.generic_engine_model_a.earlycollect.bank-v1+json": {
      "action": "earlycollect_bank",
      "$schema": "http://json-schemas.geco.io/slots/earlycollect-v1.json"
    },
    "application/vnd.miko.slots.generic_engine_model_a.earlycollect.collect-v1+json": {
      "action": "earlycollect_collect",
      "perLine": true,
      "$schema": "http://json-schemas.geco.io/slots/earlycollect-v1.json",
      "stake": null,
      "selectedWinLines": []
    },
    "application/vnd.geco.slots.spin-v1+json": {
      "action": "spin",
      "perLine": true,
      "$schema": "http://json-schemas.geco.io/slots/spin-v1.json",
      "stake": null,
      "selectedWinLines": []
    }
  },
  "gamestate": {
    "stake": 0,
    "win": 2.1,
    "action": "interlevel",
    "currentPlay": 0,
    "currentWinnings": 2.1,
```

## Links
**_links** is used as shortcut button that connects to **_forms** via `type` field value. It looks like this:
```json
 "_links": [
    {
      "href": "http://localhost:6543/game/10?ccy=GBP",
      "type": "application/vnd.miko.slots.generic_engine_model_a.earlycollect.bank-v1+json",
      "method": "POST",
      "rel": "earlycollect_bank"
    },
    {
      "href": "http://localhost:6543/game/10?ccy=GBP",
      "type": "application/vnd.miko.slots.generic_engine_model_a.earlycollect.collect-v1+json",
      "method": "POST",
      "rel": "earlycollect_collect"
    },
    {
      "href": "http://localhost:6543/gameplay/27038/client_state",
      "type": "application/octet-stream",
      "method": "PUT",
      "rel": "gameplay-client-state-save"
    }
  ],
```
Also, there are clickable buttons shown in debug page that will automatically copy and past the required "form" to request body field. For example, you should see something like this in request body frame after clicking "early_collect (POST)" button: _(caution: if you see different things, please ask to redeploy generic backend)_
```json
{
  "action": "earlycollect_collect",
  "perLine": true,
  "$schema": "http://json-schemas.geco.io/slots/earlycollect-v1.json",
  "stake": "0.04",
  "selectedWinLines": [
    "0",
    "1",
    "2",
    "3",
    "4",
    "5",
    "6",
    "7",
    "8",
    "9"
  ]
}
```
you do not need manully assign values to each field shown above as they have values in default. eg. `"stake":"0.04",`. However, sometime you have to manully fill some fields with your own values. For example, after clicking button of "Intervelevl", you see this in request body frame:
```json
{
  "action": "interlevel",
  "$schema": "http://json-schemas.geco.io/slots/interlevel-v1.json"
}
```
But, the one shown in *_forms* is looking like this:
```json
  "_forms": {
    "application/vnd.miko.slots.generic_engine_model_a.interlevel-v1+json": {
      "action": "interlevel",
      "cashShipsCollected": [],
      "$schema": "http://json-schemas.geco.io/slots/interlevel-v1.json",
      "energyShipsCollected": []
    }
  },
```
Obviously,`"energyShipsCollected": []` is missing and so you have to manully add it and assign values (here they are ships player catches) in request body frame. after that, it should look like this (assume player catches all energy ships):
```json
{
  "action": "interlevel",
  "$schema": "http://json-schemas.geco.io/slots/interlevel-v1.json",
  "cashShipsCollected":[],
  "energyShipsCollected":["ep1","ep2","ep3"]
}
```
As you can see, `"energyShipsCollected":["ep1","ep2","ep3"]` has fixed value like ep1 ep2 ep3. All possible manully -assigned-values are listed below:  
"energyShipsCollected": ["ep1","ep2","ep3"] // player catches all energy ships  
"energyShipsCollected": ["ep1"] // player catches not all energy ships  
"energyShipsCollected": [] // empty means player catches no energy ships  
"cashShipsCollected": ["cash1","cash2","cash3"] // player catches all cash ships  
"cashShipsCollected": ["cash1"] // player catches not all cash ships  
"cashShipsCollected": [] // empty means player catches no cash ships  

 



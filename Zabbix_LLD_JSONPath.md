OK folks... I've been working on this forever and FINALLY have it working!! Strap in as this is going to be a bit. 

## Overall Process using JSONPath
1. Create a HTTP Agent item to query the API
2. Create Discovery Rule --> Type: Dependent Item --> Get Gateway Names
3. Create Item Prototype --> Type: Dependent Item --> Get Gateway Status

### Create a HTTP Agent item to query the API 

**IMPORTANT:**

Http Agent item will use BASIC Auth to connect and the "User name" / "Password" field length is limited to 64 chars. Since OPNSense makes KEY/SECRETS longer than that, we have to make {#KEY} / {#SECRET} macros to hold the longer string. Place them in their respective fields "User Name / KEY" and "Password / SECRET". 

While we're at it, we might as well do the same for the {#ROUTER_IP} or #{ROUTER_URL} and put it in the URL string like "https://{#ROUTER_IP / URL}/api/routes/gateway/status"   
 
> #### Http Agent Item  
> ***If I don't mention it here, it was default or nothing***
> * Name --> Gateway Status API Poll
> * Type: Http Agent
> * Key: opnsense.gw.status.get (***Sidenote: this doesn't matter***)
> * Type of Info: Text
> * URL: https://{#ROUTER_IP / URL}/api/routes/gateway/status
> * Request: Type: GET
> * Request Body Type: Raw
> * HTTP Auth: Basic ---> **See Above**
> * History Storage Period: *Optional, but I turned it down to lowest possible (1d) just to see the data coming in*

Using the above setup, let it test and make sure you get data back. This is what I got....minus the ACTUAL IPs #notanidiot 


```json
{
   "items":[
      {
         "name":"WAN_DHCP",
         "address":"11.22.33.44",
         "status":"none",
         "loss":"0.0 %",
         "delay":"11.1 ms",
         "stddev":"1.6 ms",
         "status_translated":"Online"
      },
      {
         "name":"Testing",
         "address":"192.199.100.254",
         "status":"force_down",
         "loss":"~",
         "delay":"~",
         "stddev":"~",
         "status_translated":"Offline (forced)"
      }
   ],
   "status":"ok"
}
```

Good.... Onwards and Upwards!! 

## Create Discovery Rule --> Type: Dependent Item --> Get Gateway Names

 What we're wanting to do here is to have Zabbix look at the above JSON and pull the "name" items. Since I can't "test" using the macro section, I used the "Preprocessing" to just test and then move the path to a macro.
 
 > #### Discovery Rule
 > ***If I don't mention it here, it was default or nothing***
 > * Name: Discover Gateway Names
 > * Type: Dependent Item
 > * Key: opnsense.gw.dsc (***Sidenote: Again ... doesn't matter***)
 > * Master Item: Select > "Gateway Status API Poll"
 
 > ##### Preprocessing Rule 
 > * Create a JSONPath step and have it look at -->  $.items  
 > * Test all steps --> Copy JSON to value from the previous Http Agent Item --> Test.   
 >   
 >  You should get back something like. 
 
    
```json
[
    {
        "name":"WAN_DHCP",
        "address":"XX.XX.XX.XX", #notanidiot
        "status":"none",
        "loss":"0.0 %",
        "delay":"11.1 ms",
        "stddev":"2.1 ms",
        "status_translated":"Online"
    },
    {
        "name":"Testing",
        "address":"XX.XX.XX.XX",
        "status":"force_down",
        "loss":"~",
        "delay":"~",
        "stddev":"~",
        "status_translated":"Offline (forced)"
    }
]
    
```
 
 > ##### TESTING the MACRO ---> Optional but this is how I test sometimes
 > * Create a second JSONPath step AFTER $.items and have it look at -->  $..name  
 > * Test all steps --> Copy JSON to value from the previous Http Agent Item --> Test. 
 >
 > You should get back something like...
 ```json
 ["WAN_DHCP","Testing"] 
 ```
 > Cool, now we know what exactly we'll get back from $..name. Soo...
 > * Delete the 2nd JSONPath ($..name) and create it as a LLD Macro
 >    * {#GWNAME} ---> JSONPath ---> $..name
 >
 
 MOVING ON PEOPLE!!!! 
 
## Create Item Prototype --> Type: Dependent Item --> Get Gateway Status

OK, you made it this far, now welcome to the land of the PITA. I think here is where most everyone (including me) started pulling our hair out. I won't go over ALL the errors that we've probably all seen. #toomanybadmemories   
  
> ### Item Proto... WAIT!!!

> ### Macro Functions (aka. The missing link)

Yes, the problem is that we need to clean up the {#GWNAME} just a bit
See, it's not getting JUST WAN_DHCP....nope, it's getting ["WAN_DHCP"] and those brackets and quotes are sending us straight to the insane asylum. Thankfully the good people Zabbix saw fit to create functions that can act on macros PRE evaluation. (Although, I wish they would have made them a bit easier to find)

We need to change {#GWNAME} to ..... THIS {{#GWNAME}.regsub("\\[\\" ([A-Z].*)\\"\\]", \\1)}

Huh?? ... Chu talkin' bout willis ??? 

What this does is look at the macro and strips unwanted chars out. In this case the [ ] and ". 

When we use this we now have WAN_DHCP.... just like we want. 

*We now return you to your help....*

> ### Item Prototypes 
> * Name: {{#GWNAME}.regsub("\\[\\" ([A-Z].*)\\"\\]", \\1)} Status ---> (AKA: WAN_DHCP Status)
> * Type: Dependent item
> * Key: opnsense.gw.status[{#GWNAME}] --> ***Odd thing, I couldn't put a macro here, but it still works***
> * Type of Info: Text
> * Master Item: Gateway Status API Poll
> * Storage Period: 90d
> * Trend Storage: 1y
>
>
> ##### Pre-processing Steps
> * JSONPath: $.items[?(@.name == "{{#GWNAME}.regsub("\\[\\" ([A-Z].*)\\"\\]", \\1)}")].status_translated --> ***Yes, we filtered out $.items previously, however, this is looking at the "Master Item" that HASN'T been filtered***
> * Trim: ["]

The JSONPath will look at the Master Item and search for the status_translated property where the name equals our modified macro {#GWNAME} IE: WAN_DHCP

....We then trim for brackets and quotes to make it look nice. 

Whew...Made it.... Give it some time to populate and you *should* start seeing the correct values. 







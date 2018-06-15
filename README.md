# OLI-Trigger-SFDC
All OLI triggers in SFDC


Trigger 1

"
  public void OnBeforeInsert(List<Order_Line_Item__c> newList){
    processSOPFields(null, newList);
    copyOrderFieldToOLIfield(newList);
    copyRFPCaseToOLIField(newList);
    //validateOLIStartEndDates(null, newList);
    validateReqdFields(newList);
    //validateSOPCampaignURL(newList); 
    calculateFields(newList);
    setDefaultOLIFields(newList);"
    
    Trigger 2
    
    " public void OnBeforeUpdate(Map<ID, Order_Line_Item__c> oldMap, List<Order_Line_Item__c> newList){
    autoPopulateOLIName(newList, oldMap);
    processSOPFields(oldMap, newList);
    validateReqdFields(newList);
    //validateOLIStartEndDates(oldMap, newList);
    //validateSOPCampaignURL(newList);
    calculateFields(newList);
    setDefaultOLIFields(newList);
    checkIfOLIWasExtended(oldMap, newList);
    processNetsuiteFields(oldMap, newList);"
    
    
    Trigger 3
    
    "  public void OnAfterInsert(Map<ID, Order_Line_Item__c> oldMap, Map<ID, Order_Line_Item__c> newMap, Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
    //createMSR(newORderLineItems, 'insert', null);
    //createTwitLinkedInCase(newOrderLineItems);
    //createCreditCheckCase(newOrderLineItems);
    createCaseOnInsert(updatedOrderLineItems);
    
      //processDSOEmail(newMap);
        validateOLIStartEndDates(oldMap, updatedOrderLineItems);
        if (Test.isRunningTest())  testCovFiller();"
        
     Trigger 4
     
     " //public void OnAfterUpdate(Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems, Map<ID, Order_Line_Item__c> oldOrderMap, Map<ID, Order_Line_Item__c> newOrderMap){
  //  if(firstRun){
  //    firstRun = false;
  //    updateOrderLineItemRates(newOrderMap.keySet());
  //  }"
  
  Trigger 5
  
  "  public void OnAfterUpdateNew(Map<ID, Order_Line_Item__c> oldMap, Map<ID, Order_Line_Item__c> newMap, Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
    if(firstRun){
      firstRun = false;
      createCase(oldOrderLineItems, updatedOrderLineItems, oldMap);
      createConfirmBusinessLocCase(oldOrderLineItems, updatedOrderLineItems);
      //updateOrderExtendedStatusToLive(oldMap, oldOrderLineItems, updatedOrderLineItems);   - replaced by updateOrderStatus()
      //updateOrderStatusToDelivered(oldOrderLineItems, updatedOrderLineItems);       - replaced by updateOrderStatus()
      updateOrderStatus(oldMap, oldOrderLineItems, updatedOrderLineItems);
      makeUPCalloutIfCMUpdated(oldMap, oldOrderLineItems, updatedOrderLineItems);
      
      //processDSOEmail(newMap);
      validateOLIStartEndDates(oldMap, updatedOrderLineItems);
      //createMSR(updatedOrderLineItems, 'update', oldMap);
    }    
  }"
  
  Trigger 6
  
  "public void OnAfterInsertUpdate(Map<ID, Order_Line_Item__c> oldMap, Map<ID, Order_Line_Item__c> newMap, Order_Line_Item__c[] updatedOrderLineItems){
   // processDSOEmail(newMap);
    //validateOLIStartEndDates(oldMap, updatedOrderLineItems);
  }"
  
  
  Trigger 7
  
  "private void makeUPCalloutIfCMUpdated(Map<ID, Order_Line_Item__c> oldMap, Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
        
        Set<Id> oliIdSet = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            if (updatedOrderLineItems[i].Order__c!=null)    oliIdSet.add(updatedOrderLineItems[i].Order__c);
        }
        
        List<Order__c> orderList = [select Status__c, Start_Date__c, End_Date__c, Id from Order__c where Id IN :oliIdSet];
        Map<Id,Order__c> orderMap = new Map<Id,Order__c>();
        for (Order__c o : orderList){
            orderMap.put(o.Id,o);
        }
        
        SOP2_Integration__c mc = SOP2_Integration__c.getOrgDefaults();
        
        for(Order_Line_Item__c oli : updatedOrderLineItems){
          Order_Line_Item__c tempOldOLI = oldMap.get(oli.Id);
            if (tempOldOLI.Order_CM_Email__c != oli.Order_CM_Email__c && oli.Order_CM_Email__c!=null &&
              oli.SOP2_Initiative_Line_Item_ID__c!=null && oli.SOP2_Initiative_Line_Item_ID__c!='' &&
              mc.allow_integration__c)  
                SOP2_RestService.sop2UpdateInitiativeLineItemCreatorCalloutOLILevel(oli.Id, oli.SOP2_Initiative_Line_Item_ID__c, oli.Order_CM_Email__c, null);
            
        }
        
    }
  "
  
  Trigger 8
  
  "public void setDefaultOLIFields(List<Order_Line_Item__c> newOLIList){
    
    Set<Id> fbaaIdSet = new Set<Id>();
    for (Order_Line_Item__c newOLI : newOLIList){
        if (newOli.Publisher_Ad_Account__c!=null && (newOLI.Platform__c == 'Facebook' || newOLI.Platform__c == 'Twitter' || newOLI.Platform__c == 'Instagram'))
            fbaaIdSet.add(newOli.Publisher_Ad_Account__c);
        if (newOLI.Platform__c == 'LinkedIn')  newOLI.Not_Supported_via_Api__c = true; // BOSP-411 and BOSP-424
        // removed newOLI.Platform__c == 'Pinterest' || BOSP-639
    }
    
    Map<Id,Publisher_Ad_Account__c> fbAddAcctMap = new Map<Id,Publisher_Ad_Account__c>();
    for (Publisher_Ad_Account__c fbaa : [select Id, Publisher_Direct_Bill__c 
              from Publisher_Ad_Account__c where ID IN :fbaaIdSet]){
        fbAddAcctMap.put(fbaa.Id,fbaa);
    }
    
    for (Order_Line_Item__c newOLI : newOLIList){
      if (newOli.Publisher_Ad_Account__c!=null && (newOLI.Platform__c == 'Facebook' || newOLI.Platform__c == 'Twitter' || newOLI.Platform__c == 'Instagram')){
        Publisher_Ad_Account__c fbaa = fbAddAcctMap.get(newOli.Publisher_Ad_Account__c);
        newOLI.Publisher_Direct_Bill__c = fbaa.Publisher_Direct_Bill__c;
      }
      
      if (newOLI.Make_Good__c == true ||
        newOLI.OLI_Budget_is_partially_MG_Media__c == true)
          newOLI.Make_Good_Occurred__c = true;
      else  newOLI.Make_Good_Occurred__c = false;
    }
  }"
  
  Trigger 9
  
  " public void autoPopulateOLIName(List<Order_Line_Item__c> newOLIList, Map<ID, Order_Line_Item__c> oldMap){
    
    Set<Id> orderIdSet = new Set<Id>();
    Set<Id> fbaaIdSet = new Set<Id>();
    for (Order_Line_Item__c newOLI : newOLIList){
        if (newOli.Order__c!=null)
            orderIdSet.add(newOli.Order__c);
    }
    Map<Id,Order__c> orderMap = new Map<Id,Order__c>();
    for (Order__c o : [select Id, Start_Date__c, End_Date__c, Billable_Advertiser_Primary_Contact__c, Opportunity__r.Opportunity_Primary_Contact__c,
                Billable_Advertiser__r.Name,
                Senior_Account_Manager__c, Senior_Account_Manager__r.Name
              from Order__c where ID IN :orderIdSet]){
        orderMap.put(o.Id,o);
    }
    
    for (Order_Line_Item__c newOLI : newOLIList){
      if (newOli.Order__c!=null){
            if (orderMap.containsKey(newOli.Order__c)){
                Order__c o = orderMap.get(newOli.Order__c);
                
                String autoPopulateOLIName = '';
                
                if (newOLI.Product__c=='SOP' || newOLI.Product__c=='Intelligence'){
                  autoPopulateOLIName = '' + ((o.Billable_Advertiser__r.Name!=null && o.Billable_Advertiser__r.Name!= '') ? o.Billable_Advertiser__r.Name : '') + ' - ' +  ((newOli.Platform__c!=null && newOli.Platform__c!= '') ? newOli.Platform__c : '') + ' - ' +  ((newOli.Tactics__c!=null && newOli.Tactics__c!= '') ? newOli.Tactics__c : '') + ' - ' +  ((newOli.Unified_Fee__c!=null) ? String.valueOf(newOli.Unified_Fee__c) : '');
          
                }else{
                  autoPopulateOLIName = '' + ((newOli.Strategy_Goal__c!=null && newOli.Strategy_Goal__c!= '') ? newOli.Strategy_Goal__c : '');// + ' - ' +  ((newOli.Client_Reference_ID__c!=null && newOli.Client_Reference_ID__c!= '') ? newOli.Client_Reference_ID__c : '') ;
            
                }
                if (autoPopulateOLIName.length()>80)  autoPopulateOLIName = autoPopulateOLIName.substring(0,80);
                if (newOli.Name==null || newOli.Name=='' || newOli.Name=='Order Line Item Name')  {// ||
                //  (oldMap.get(newOli.Id).Strategy_Goal__c!=newOli.Strategy_Goal__c && newOli.Strategy_Goal__c!=null)  ){
                
                  newOli.Name = autoPopulateOLIName;
                  //newOli.Strategy_Goal__c = autoPopulateOLIName;
              }
              
              //if (newOli.Strategy_Goal__c == 'Order Line Item Name')  newOli.Strategy_Goal__c = autoPopulateOLIName;
              
                if (o.Senior_Account_Manager__c!=null)  newOli.Senior_Account_Manager__c = o.Senior_Account_Manager__r.Name;
                
            }
                
                
            }
      }
  }"
  
  Trigger 10
  
  "public void syncToSOPCallout(Map<ID, Order_Line_Item__c> oldMap, Map<ID, Order_Line_Item__c> newMap, Order_Line_Item__c[] updatedOrderLineItems){
    Integer ctr = 0;
    for (Order_Line_Item__c newOLI : updatedOrderLineItems){
      Order_Line_Item__c oldOLI = oldMap.get(newOLI.Id);
      
      if ( ctr < 10 && 
        (  (oldOLI.isDeployedToSOP1__c==false && newOli.isDeployedToSOP1__c==true) || 
          (newOli.isDeployedToSOP1__c && 
            (  oldOLI.Strategy_Goal__c != newOLI.Strategy_Goal__c ||
            oldOLI.Product__c != newOLI.Product__c ||
            oldOLI.Brand__c != newOLI.Brand__c ||
            oldOLI.Agency__c != newOLI.Agency__c ||
            oldOLI.Start_Date__c != newOLI.Start_Date__c ||
            oldOLI.End_Date__c != newOLI.End_Date__c ||
            oldOLI.Billing_Rate_Type__c != newOLI.Billing_Rate_Type__c ||
            oldOLI.Billing_Rate__c != newOLI.Billing_Rate__c ||
            oldOLI.Weird_SOP_Rate__c != newOLI.Weird_SOP_Rate__c ||
            oldOLI.Estimated_Rate_Per_Unit__c != newOLI.Estimated_Rate_Per_Unit__c ||
            oldOLI.Estimated_Imp_Clicks_Actions__c != newOLI.Estimated_Imp_Clicks_Actions__c ||
            oldOLI.Platform__c != newOLI.Platform__c ||
            oldOLI.Tactics__c != newOLI.Tactics__c ||
            oldOLI.Estimated_Rate_Type__c != newOLI.Estimated_Rate_Type__c ||
            oldOLI.Total_Advertiser_Cost__c != newOLI.Total_Advertiser_Cost__c ||
            oldOLI.Id != newOLI.Id ||
            oldOLI.Sub_Tactics__c != newOLI.Sub_Tactics__c ||
            oldOLI.Planner__c != newOLI.Planner__c ||
            //oldOLI.SOP_Integration_ID__c != newOLI.SOP_Integration_ID__c ||
            //oldOLI.SOP_Campaign_ID__c != newOLI.SOP_Campaign_ID__c ||
            //oldOLI.SOP_Spend_Type__c != newOLI.SOP_Spend_Type__c ||
            //oldOLI.SOP_Conversion_Goal__c != newOLI.SOP_Conversion_Goal__c ||
            oldOLI.Publisher_Direct_Bill_To_Advertiser__c != newOLI.Publisher_Direct_Bill_To_Advertiser__c))) &&
          newOLI.isDeployedToSOP1__c == true){
      String method = ((oldOLI.isDeployedToSOP1__c==false && newOLI.SOP_Campaign_ID__c==null) && (newOLI.isDeployedToSOP1__c==true)) ? 'POST' : 'PUT';
      //String method = (oldOLI.isCaseCreatedViaDeploy__c==false) ? 'POST' : 'PUT';
      
      String mainJSON = SOP_RestService.createJSONString(oldOLI, newOLI, method);
      //if (!(oldOLI.isCaseCreatedViaDeploy__c==false && newOLI.isCaseCreatedViaDeploy__c==true && newOLI.SOP_Campaign_ID__c!=null)) 
        SOP_RestService.sopSyncCallout(mainJSON, method);
      ctr++;
    }
    }
  }"
  
  Trigger 11
  
  "public void syncToSOP2Callout(Map<ID, Order_Line_Item__c> oldMap, Map<ID, Order_Line_Item__c> newMap, Order_Line_Item__c[] updatedOrderLineItems){
    Integer ctr = 0;
    LisT<Order_Line_Item__c> oliToSyncList = new List<Order_Line_Item__c>();
    for (Order_Line_Item__c newOLI : updatedOrderLineItems){
      Order_Line_Item__c oldOLI = oldMap.get(newOLI.Id);
      
      // only sync to SOP2 if status is Deployed or Live
      if ((newOLI.Status__c == 'Delayed' || newOLI.Status__c == 'Deployed' || newOLI.Status__c == 'Live' || newOLI.Status__c == 'Delivered' || newOLI.Status__c == 'Canceled') && 
        (oldOLI.SOP2_Initiative_Line_Item_ID__c!=null && oldOLI.SOP2_Initiative_Line_Item_ID__c!='' && oldOLI.SOP2_Initiative_Line_Item_ID__c!='null') &&
        (newOLI.SOP2_Initiative_Line_Item_ID__c!=null && newOLI.SOP2_Initiative_Line_Item_ID__c!='' && newOLI.SOP2_Initiative_Line_Item_ID__c!='null') &&
        (oldOLI.Strategy_Goal__c != newOLI.Strategy_Goal__c ||
        oldOLI.Start_Date__c != newOLI.Start_Date__c ||
        oldOLI.End_Date__c != newOLI.End_Date__c ||
        oldOLI.Budget__c != newOLI.Budget__c ||
        oldOLI.Billing_Rate_Type__c != newOLI.Billing_Rate_Type__c ||
        oldOLI.Billing_Rate__c != newOLI.Billing_Rate__c ||
        oldOLI.Unified_Fee__c != newOLI.Unified_Fee__c ||
        oldOLI.Total_Advertiser_Cost__c != newOLI.Total_Advertiser_Cost__c) &&
        newOLI.isDeployedToSOP2__c == true){
        oliToSyncList.add(newOLI);
      }
    }
    
    if(oliToSyncList.size()>0){
      // make callout for synching Initiative on SOP2 here
        Map<String,String> recIdJSONStringMap = new Map<STring,String>();
        //if (oli.SOP2_Initiative_Line_Item_ID__c!=null && oli.SOP2_Initiative_Line_Item_ID__c!=''){
          recIdJSONStringMap = SOP2_RestService.createSOP2OrderJSONString(null, oliToSyncList);
          SOP2_RestService.sop2DeployOrderCallout(recIdJSONStringMap);
    //}
    }
    //if (oldOLI.Total_Advertiser_Cost__c   != newOLI.Total_Advertiser_Cost__c)  System.debug('======= sisiw: Total_Advertiser_Cost__c updated' + newOLI.Total_Advertiser_Cost__c);
  
  }
"

Trigger 12

"private void validateReqdFields(List<Order_Line_Item__c> newOLIList){
      for (Order_Line_Item__c oli : newOLIList){
        
      if (oli.Data_Source_Types__c != null && oli.Data_Source_Types__c != ''){
          String [] dsCount = oli.Data_Source_Types__c.split(';');
          oli.Data_Sources__c = dsCount.size();
      }else oli.Data_Sources__c = 0;
      
          if (oli.Tactics__c == 'Standard Intelligence Reports' && (oli.Social_Profiles__c==null))
                oli.addError('Social Profiles is a required field for Standard Intelligence Reports. Please provide details.');
          
          if (oli.Tactics__c == 'Custom Intelligence Reports' && (oli.Data_Source_Types__c == null || oli.Data_Source_Types__c == '') )
              oli.Data_Source_Types__c.addError('Data Source Types is a required field for Custom Intelligence Reports. Please provide details.'); 
        
          Set<String> subTactics = new Set<String>();
          if (oli.Sub_Tactics__c!=null){
              for (String s : oli.Sub_Tactics__c.split(';')){
                subTactics.add(s);
              }
          }
          if (oli.Platform__c == Constants.PLATFORM_UNIFIED && 
             (oli.Tactics__c == 'SOP' || oli.Tactics__c == 'SOP - Light' || oli.Tactics__c == 'SOP - Scale') && 
             (oli.Sub_Tactics__c == 'Owned' || subTactics.contains('Owned')) &&
             (oli.Symphony_Package__c == null || oli.Symphony_Package__c == '')){
              oli.Symphony_Package__c.addError('Please indicate a Symphony Package value.');
           
            }
            
            if(oli.Platform__c == Constants.PLATFORM_UNIFIED && oli.Number_of_Users__c == null && !Test.isRunningTest()){
            oli.Number_of_Users__c.addError('This is required for ' + Constants.PLATFORM_UNIFIED + '. Please specify the number of users.');
          }
          
          // BOSP-467 and BOSP-535
          if ( (oli.Make_Good__c == true || oli.OLI_Budget_is_partially_MG_Media__c == true) &&
             (oli.Make_Good_Case_Link__c==null ||
             oli.Credit_Fee_Amount__c==null || oli.Credit_Fee_Amount__c==0)){
               oli.addError('Please enter the Make Good Case Number and Credit Fee Amount');
           }
           
        }
    }"
    
  Trigger 13
  
  "  private void createCaseOnInsert(Order_Line_Item__c[] newOrderLineItems){
    List<Case> caseToInsert = new List<Case>();
    Map<Id,Id> oliIdOrderIdMap = new Map<Id,Id>();
    Map<Id,Order__c> orderIdOrderMap = new Map<Id,Order__c>();
    for (Order_Line_Item__c oli : newOrderLineItems){
      if (oli.Platform__c!=null && (oli.Platform__c.equalsIgnoreCase('Snapchat') || oli.Platform__c.equalsIgnoreCase('Twitter') || oli.Platform__c.equalsIgnoreCase('LinkedIn') || oli.Platform__c.equalsIgnoreCase('Facebook') || oli.Platform__c.equalsIgnoreCase('Instagram')) && oli.Order__c!=null){
        oliIdOrderIdMap.put(oli.Id, oli.Order__c);
      }
    }
    for (Order__c o : [select Id, Opportunity__r.AccountId, Opportunity__c, Account_Manager__c, Campaign_Manager__c from Order__c where Id IN :oliIdOrderIdMap.values()]){
      orderIdOrderMap.put(o.Id, o);
    }"
  
  Trigger 14
  
  " //RecordType co = [Select Id From RecordType Where Name='Campaign Management'];
    for (Order_Line_Item__c oli : newOrderLineItems){
      if (oli.Platform__c!=null && (oli.Platform__c.equalsIgnoreCase('Twitter') || oli.Platform__c.equalsIgnoreCase('LinkedIn')) && oli.Order__c!=null && orderIdOrderMap.containsKey(oli.Order__c)){
        Order__c tempOrder = orderIdOrderMap.get(oli.Order__c);
        
        String oliName = '' + ((oli.Strategy_Goal__c!=null && oli.Strategy_Goal__c!= '') ? oli.Strategy_Goal__c : '') + ' - ' +  ((oli.Client_Reference_ID__c!=null && oli.Client_Reference_ID__c!= '') ? oli.Client_Reference_ID__c : '') ;
        if (oliName.length()>80)  oliName = oliName.substring(0,80);
        Case c = new Case(  Subject = 'Create Publisher IO for ' + oliName,
                  Order_Line_Item__c = oli.ID,
                  Opportunity__c = tempOrder.Opportunity__c,
                  AccountId = tempOrder.Opportunity__r.AccountId, 
                  Assigned_To__c = tempOrder.Account_Manager__c,
                  Order__c = oli.Order__c,
                  RecordTypeId = OrderLineItemTriggerHandler.cmCaseRecTypeId,
                        Area__c = 'Campaign Management',
                        Sub_Area__c = 'Create Publisher IO',
                        Description = 'Create Publisher IO for ' + oliName);
                  
        c = copyOliToCaseFields(c, oli);
        caseToInsert.add(c);
      
      }else if (oli.Platform__c!=null && (oli.Platform__c.equalsIgnoreCase('Snapchat') || oli.Platform__c.equalsIgnoreCase('Facebook') || oli.Platform__c.equalsIgnoreCase('Instagram')) && oli.Order__c!=null && orderIdOrderMap.containsKey(oli.Order__c)){
      
        Order__c tempOrder = orderIdOrderMap.get(oli.Order__c);
        
        String oliName = '' + ((oli.Strategy_Goal__c!=null && oli.Strategy_Goal__c!= '') ? oli.Strategy_Goal__c : '') + ' - ' +  ((oli.Client_Reference_ID__c!=null && oli.Client_Reference_ID__c!= '') ? oli.Client_Reference_ID__c : '') ;
        if (oliName.length()>80)  oliName = oliName.substring(0,80);
        Case c = new Case(  Subject = 'Credit check needed for ' + oli.Brand__c + ' spend account',
                      Order_Line_Item__c = oli.ID,
                      Opportunity__c = tempOrder.Opportunity__c,
                      AccountId = tempOrder.Opportunity__r.AccountId, 
                      
                      Order__c = oli.Order__c,
                      RecordTypeId = OrderLineItemTriggerHandler.cmCaseRecTypeId,
                      Area__c = 'Campaign Management',
                      Sub_Area__c = 'Spend Account Credit Check');
    
    if (oli.Platform__c.equalsIgnoreCase('Facebook'))    c.Description = 'A new Facebook line item has been created, please check the spend account for available credit. If the account is low on credit please reach out to Facebook to increase the credit (CC fbaccountcredit@unified.com). ';
        else if (oli.Platform__c.equalsIgnoreCase('Instagram'))  c.Description = 'A new Instagram line item has been created, please check the spend account for available credit. If the account is low on credit please reach out to Instagram to increase the credit (CC fbaccountcredit@unified.com).';
        else if (oli.Platform__c.equalsIgnoreCase('Snapchat'))  c.Description = 'A new Snapchat line item has been created, please check the spend account for available credit. If the account is low on credit please reach out to Snapchat to increase the credit (CC fbaccountcredit@unified.com).';
        
        if (tempOrder.Account_Manager__c!=null)    c.OwnerId = tempOrder.Account_Manager__c;   
        if (tempOrder.Campaign_Manager__c!=null)  c.Assigned_To__c = tempOrder.Campaign_Manager__c;    
        
        c = copyOliToCaseFields(c, oli);
        caseToInsert.add(c);
      }"
      
     Trigger 15
     
     " 
      if (oli.Platform__c!=null && oli.Platform__c.equalsIgnoreCase('Snapchat') && oli.Order__c!=null && orderIdOrderMap.containsKey(oli.Order__c)){
        Order__c tempOrder = orderIdOrderMap.get(oli.Order__c);
        
        String oliName = '' + ((oli.Strategy_Goal__c!=null && oli.Strategy_Goal__c!= '') ? oli.Strategy_Goal__c : '') + ' - ' +  ((oli.Client_Reference_ID__c!=null && oli.Client_Reference_ID__c!= '') ? oli.Client_Reference_ID__c : '') ;
        if (oliName.length()>80)  oliName = oliName.substring(0,80);
        Case c = new Case(  Subject = 'Create Snapchat Ad Account for ' + oliName,
                  Order_Line_Item__c = oli.ID,
                  Opportunity__c = tempOrder.Opportunity__c,
                  AccountId = tempOrder.Opportunity__r.AccountId, 
                  Assigned_To__c = tempOrder.Account_Manager__c,
                  Order__c = oli.Order__c,
                  RecordTypeId = OrderLineItemTriggerHandler.cmCaseRecTypeId,
                        Area__c = 'Campaign Management',
                        Sub_Area__c = 'Create Snapchat Ad Account',
                        Description = 'PLEASE Create Snapchat Ad Account for' + oliName);
                  
        c = copyOliToCaseFields(c, oli);
        caseToInsert.add(c);
      
      }
    }
    if (caseToInsert.size()>0)  insert caseToInsert;
  }"
  
  Trigger 16
  
  "/* Called on before update to see if oli was extended, if yes, set status from Delivered to become Live  BOSP-315 */ 
    private void checkIfOLIWasExtended(Map<ID, Order_Line_Item__c> oldMap, List<Order_Line_Item__c> newOLIList){
        
        Set<Id> oliIdSet = new Set<Id>();
        for(Order_Line_Item__c oli : newOLIList){
          if (oldMap!=null && // means it is an update event
            oldMap.get(oli.Id).End_Date__c!=oli.End_Date__c && // means the end date value was changed
            oldMap.get(oli.Id).End_Date__c<oli.End_Date__c && // means the new end date is later than the old value
            oli.Status__c == 'Delivered'){
            oli.Status__c = 'Live';
          }
          
          // BOSP-504
          if (oldMap!=null && // means it is an update event
            oldMap.get(oli.Id).Start_Date__c!=oli.Start_Date__c && // means the start date value was changed
            (oli.SOP2_Initiative_Line_Item_ID__c==null ||
             oli.SOP2_Initiative_Line_Item_ID__c=='') && 
             oli.Start_Date__c >= Date.today() && // means the new start date is later than today
             oli.Status__c == 'Delayed'){
            
            oli.Status__c = 'New';
          }
        }
    }"
    
    Trigger 17
    
    " /*
        If insert, check if the start date is earlier than the start date of the related order, if yes, update the order to the same start date
        If update, check if the end date is later than the end date of the related order, if yes, update the order to the same end date
    */
    private void validateOLIStartEndDates(Map<ID, Order_Line_Item__c> oldMap, List<Order_Line_Item__c> newOLIList){
        
        Set<Id> oliIdSet = new Set<Id>();
        for(Order_Line_Item__c oli : newOLIList){
            if (oli.Order__c!=null){
              if (  oldMap==null || // means it is an insert event
                  (oldMap!=null && // means it is an update event
                   oldMap.get(oli.Id).Start_Date__c!=oli.Start_Date__c ||
                   oldMap.get(oli.Id).End_Date__c!=oli.End_Date__c ||
                   (oldMap.get(oli.Id).Status__c!=oli.Status__c && oli.Status__c == 'Canceled') ||
                   (oldMap.get(oli.Id).Status__c!=oli.Status__c && oldMap.get(oli.Id).Status__c == 'Canceled')
                  ) 
                ){
                oliIdSet.add(oli.Order__c);
              }
            }
        }
        
        if (oliIdSet.size()>0){
          Map<Id,Order__c> orderMap = new Map<Id,Order__c>();
          for (Order__c o : [select Status__c, Start_Date__c, End_Date__c, Id from Order__c where Id IN :oliIdSet]){
              orderMap.put(o.Id,o);
          }
          
          if (orderMap.size() > 0)   update orderMap.values();
        }
       
    }"
    
    Trigger 18
    
    "/*  When an Order Line Item Status is set to Completed, check all other sibling OLI under an order, if all are Completed, set the Order status to 'Delivered' */
    private void updateOrderStatusToDelivered(Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
        
        Set<Id> oliOrderIdSet = new Set<Id>();
        Set<Id> oliIdSet = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            oliIdSet.add(updatedOrderLineItems[i].Id);
            if (updatedOrderLineItems[i].Order__c!=null && 
                oldOrderLineItems[i].Status__c!= updatedOrderLineItems[i].Status__c && 
                updatedOrderLineItems[i].Status__c == 'Completed')    oliOrderIdSet.add(updatedOrderLineItems[i].Order__c);
        }
        
        List<Order__c> orderList = [select Status__c, Id, 
                                        (select Status__c, Id from Order_Line_Items__r 
                                        where Status__c != 'Completed'
                                        and Id NOT IN :oliIdSet) 
                                    from Order__c where Id IN :oliOrderIdSet];
        List<Order__c> orderToUpdateList = new List<Order__c>();
        for (Order__c o : orderList){
            System.debug('sisiw=====: '+o.Order_Line_Items__r.size());
            if (o.Order_Line_Items__r.size() == 0){
                o.Status__c = 'Delivered';
                orderToUpdateList.add(o);
            }
        }        
        
        if (orderToUpdateList.size() > 0)   update orderToUpdateList;
        
    }"
    
    Trigger 19
    
    " /*  When an Order Line Item End Date is extended within the date range of the Order, update Order status to 'Live' */
    private void updateOrderExtendedStatusToLive(Map<ID, Order_Line_Item__c> oldMap, Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
        
        Set<Id> oliIdSet = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            if (updatedOrderLineItems[i].Order__c!=null)    oliIdSet.add(updatedOrderLineItems[i].Order__c);
        }
        
        List<Order__c> orderList = [select Status__c, Start_Date__c, End_Date__c, Id from Order__c where Id IN :oliIdSet];
        Map<Id,Order__c> orderMap = new Map<Id,Order__c>();
        for (Order__c o : orderList){
            orderMap.put(o.Id,o);
        }
        
        Set<Id> orderWithExtensionSet = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            Order__c tempOrder = orderMap.get(updatedOrderLineItems[i].Order__c);
            if (   ( (tempOrder.End_Date__c >= updatedOrderLineItems[i].End_Date__c &&
                    oldOrderLineItems[i].End_Date__c < updatedOrderLineItems[i].End_Date__c &&
                    updatedOrderLineItems[i].End_Date__c >= date.today()) ||
                    //(oldMap.get(updatedOrderLineItems[i].Id).SOP_Media_Cost__c != updatedOrderLineItems[i].SOP_Media_Cost__c) ) &&
                    (oldMap.get(updatedOrderLineItems[i].Id).UP_Ad_Spend__c != updatedOrderLineItems[i].UP_Ad_Spend__c) ) &&
                     updatedOrderLineItems[i].UP_Ad_Spend__c > 0 )   
                orderWithExtensionSet.add(updatedOrderLineItems[i].Order__c);
            
        }
        
        List<Order__c> orderToUpdateList = new List<Order__c>();
        for (Order__c o : orderList){
            if (orderWithExtensionSet.contains(o.Id)){
                o.Status__c = 'Live';
                orderToUpdateList.add(o);
            }
        }
        
        if (orderToUpdateList.size() > 0)   update orderToUpdateList;
        
    }
    
    // MERGED updateOrderStatusToDelivered and updateOrderExtendedStatusToLive"
    
    Trigger 20
    
    " /*  When an Order Line Item Status is set to Completed, check all other sibling OLI under an order, if all are Completed, set the Order status to 'Delivered' */
    /*  When an Order Line Item End Date is extended within the date range of the Order, update Order status to 'Live' */
    private void updateOrderStatus(Map<ID, Order_Line_Item__c> oldMap, Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
        
        Set<Id> oliOrderIdSetLive = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            if (updatedOrderLineItems[i].Order__c!=null)    oliOrderIdSetLive.add(updatedOrderLineItems[i].Order__c);
        }
        
        Set<Id> oliOrderIdSetDelivered = new Set<Id>();
        Set<Id> oliIdSet = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            oliIdSet.add(updatedOrderLineItems[i].Id);
            if (updatedOrderLineItems[i].Order__c!=null && 
                oldOrderLineItems[i].Status__c!= updatedOrderLineItems[i].Status__c && 
                updatedOrderLineItems[i].Status__c == 'Completed')    oliOrderIdSetDelivered.add(updatedOrderLineItems[i].Order__c);
        }
        
        List<Order__c> orderList = [  select Status__c, Start_Date__c, End_Date__c, Id,
                          (select Status__c, Id from Order_Line_Items__r 
                                          where Status__c != 'Completed'
                                          and Id NOT IN :oliIdSet) 
                        from Order__c 
                        where Id IN :oliOrderIdSetLive OR Id IN :oliOrderIdSetDelivered];
        Map<Id,Order__c> orderMap = new Map<Id,Order__c>();
        for (Order__c o : orderList){
            orderMap.put(o.Id,o);
        }
        
        Set<Id> orderWithExtensionSet = new Set<Id>();
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
            Order__c tempOrder = orderMap.get(updatedOrderLineItems[i].Order__c);
            if (   ( (tempOrder.End_Date__c >= updatedOrderLineItems[i].End_Date__c &&
                    oldOrderLineItems[i].End_Date__c < updatedOrderLineItems[i].End_Date__c &&
                    updatedOrderLineItems[i].End_Date__c >= date.today()) ||
                    //(oldMap.get(updatedOrderLineItems[i].Id).SOP_Media_Cost__c != updatedOrderLineItems[i].SOP_Media_Cost__c) ) &&
                    (oldMap.get(updatedOrderLineItems[i].Id).UP_Ad_Spend__c != updatedOrderLineItems[i].UP_Ad_Spend__c) ) &&
                     updatedOrderLineItems[i].UP_Ad_Spend__c > 0 )   
                orderWithExtensionSet.add(updatedOrderLineItems[i].Order__c);
            
        }
        
        List<Order__c> orderToUpdateList = new List<Order__c>();
        for (Order__c o : orderList){
            if (orderWithExtensionSet.contains(o.Id)){
                o.Status__c = 'Live';
                orderToUpdateList.add(o);
            }
        }
        
        //List<Order__c> orderToUpdateList = new List<Order__c>();
        for (Order__c o : orderList){
            if (o.Order_Line_Items__r.size() == 0 && oliOrderIdSetDelivered.contains(o.Id)){
                o.Status__c = 'Delivered';
                orderToUpdateList.add(o);
            }
        }        
        
        if (orderToUpdateList.size() > 0)   update orderToUpdateList;
        
    }
"

Trigger 22

"private void calculateFields(List<Order_Line_Item__c> newOLIList){
        for (Order_Line_Item__c newOLI : newOLIList){
          Decimal totalAdvCost = (newOLI.Total_Advertiser_Cost__c!=null) ? newOLI.Total_Advertiser_Cost__c : 0;
          Decimal mediaSpend = (newOLI.Media_Ad_Spend__c!=null) ? newOLI.Media_Ad_Spend__c : 0;
          
          newOLI.Unified_Fee2__c = totalAdvCost - mediaSpend;
        }
  }"
  
  Trigger 23
  
  " /*private void validateSOPCampaignURL(List<Order_Line_Item__c> newOLIList){
        Set<String> sopCampaignUrlSet = new Set<String>();
        for (Order_Line_Item__c newOLI : newOLIList){
            if (newOli.SOP_Campaign_URL__c!=null){
              newOli.SOP_Campaign_URL__c = newOli.SOP_Campaign_URL__c.tolowercase();
                // check that the list does not have same value of sop campaign url
                if (sopCampaignUrlSet.contains(newOli.SOP_Campaign_URL__c.replace('http://','').replace('https://',''))){
                    newOli.SOP_Campaign_URL__c.addError('Another record to be inserted has the same value. Please make them unique for each one.'); 
                }else{
                    sopCampaignUrlSet.add(newOli.SOP_Campaign_URL__c.replace('http://','').replace('https://','').tolowercase());
                }
            }
        }
        Map<String,Id> oliWithsopCampaignUrl = new Map<String,Id>();
        for (Order_Line_Item__c existingOLIList : [select Id, SOP_Campaign_URL__c 
                                                    from Order_Line_Item__c 
                                                    where SOP_Campaign_URL__c IN :sopCampaignUrlSet]){
            oliWithsopCampaignUrl.put(existingOLIList.SOP_Campaign_URL__c.replace('http://','').replace('https://',''), existingOLIList.Id);
        }
                                                    
        for (Order_Line_Item__c newOLI : newOLIList){
            if (newOli.SOP_Campaign_URL__c!=null){
              newOli.SOP_Campaign_URL__c = newOli.SOP_Campaign_URL__c.tolowercase();
                if (oliWithsopCampaignUrl.containsKey(newOli.SOP_Campaign_URL__c.replace('http://','').replace('https://',''))){
                    if (newOLI.Id != null){
                        if (oliWithsopCampaignUrl.get(newOli.SOP_Campaign_URL__c.replace('http://','').replace('https://','')) != newOLI.Id){
                            newOli.SOP_Campaign_URL__c.addError('Another record already has the same value. Please put a different one.');  
                        }
                    }else{
                        newOli.SOP_Campaign_URL__c.addError('Another record already has the same value. Please put a different one.');
                    }
                }
            }
        }
    "
    
    Trigger 24
    
    " private void copyRFPCaseToOLIfield(List<Order_Line_Item__c> newOLIList){
        Set<Id> caseIdSet = new Set<Id>();
        for (Order_Line_Item__c newOLI : newOLIList){
            if (newOli.Case__c!=null)
                caseIdSet.add(newOli.Case__c);
        }
        Map<Id,Case> caseMap = new Map<Id,Case>();
        for (Case c : [select Id, Proposal_Priority__c, Due_Date__c, Strategy_Output__c, Campaign_Name__c, Primary_Objective__c, 
                            KPIs_Success_Metrics__c, Secondary_Objective__c, Platform_request_from_client__c, Tactic_request_from_client__c
                          from Case where ID IN :caseIdSet]){
            caseMap.put(c.Id,c);
        }
        for (Order_Line_Item__c newOLI : newOLIList){
            if (newOli.Case__c!=null)
                if (caseMap.containsKey(newOli.Case__c)){
                    Case c = caseMap.get(newOli.Case__c);
                    // for additional software fields
          if (c.Proposal_Priority__c!=null)         newOli.Proposal_Priority__c = c.Proposal_Priority__c;
          if (c.Due_Date__c!=null)            newOli.Due_Date__c = c.Due_Date__c;
          if (c.Strategy_Output__c!=null)         newOli.Strategy_Output__c = c.Strategy_Output__c;
          if (c.Campaign_Name__c!=null)           newOli.Campaign_Name__c = c.Campaign_Name__c;
          if (c.Primary_Objective__c!=null)         newOli.Primary_Objective__c = c.Primary_Objective__c;
          if (c.KPIs_Success_Metrics__c!=null)      newOli.KPIs_Success_Metrics__c = c.KPIs_Success_Metrics__c;
          if (c.Secondary_Objective__c!=null)       newOli.Secondary_Objective__c = c.Secondary_Objective__c;
          if (c.Platform_request_from_client__c!=null)  newOli.Platform_request_from_client__c = c.Platform_request_from_client__c;
          if (c.Tactic_request_from_client__c!=null)    newOli.Tactic_request_from_client__c = c.Tactic_request_from_client__c;
                }
        }
    }"
    
    Trigger 25
    
    "private void copyOrderFieldToOLIfield(List<Order_Line_Item__c> newOLIList){
        Set<Id> orderIdSet = new Set<Id>();
        for (Order_Line_Item__c newOLI : newOLIList){
            if (newOli.Order__c!=null)
                orderIdSet.add(newOli.Order__c);
        }
        Map<Id,Order__c> orderMap = new Map<Id,Order__c>();
        for (Order__c o : [select Id, Start_Date__c, End_Date__c, Billable_Advertiser_Primary_Contact__c, Opportunity__r.Opportunity_Primary_Contact__c,
                    Billable_Advertiser__r.Name, Account_Manager_Email__c, Campaign_Manager_Email__c, Sales_Email__c, 
                    Senior_Account_Manager__c, Senior_Account_Manager__r.Name, Campaign_Manager__c
                  from Order__c where ID IN :orderIdSet]){
            orderMap.put(o.Id,o);
        }
        for (Order_Line_Item__c newOLI : newOLIList){
            if (newOli.Order__c!=null){
                if (orderMap.containsKey(newOli.Order__c)){
                    Order__c o = orderMap.get(newOli.Order__c);
                    
                    if (o.Campaign_Manager__c==null && !Test.isRunningTest() &&
                      String.valueOf(newOLI.recordTypeId).subString(0,15) == Constants.oliAdvertisingRecType.subString(0,15))  // for advertising oli only
                      newOLI.addError('Campaign Manager field cannot be blank on order to create an order line item');
                    
                    // populate the AM, CM, and SC email fields
                    if (o.Account_Manager_Email__c != null)  newOli.Order_AM_Email__c = o.Account_Manager_Email__c;
                    if (o.Campaign_Manager_Email__c != null) newOli.Order_CM_Email__c = o.Campaign_Manager_Email__c;
                    if (o.Sales_Email__c != null)    newOli.Order_SC_Email__c = o.Sales_Email__c;
                    
                    if(o.Senior_Account_Manager__c!=null)  newOli.Senior_Account_Manager__c = o.Senior_Account_Manager__r.Name;
                    
                    if (newOli.Planner__c==null)
                        newOli.Planner__c = o.Opportunity__r.Opportunity_Primary_Contact__c;//o.Billable_Advertiser_Primary_Contact__c;
                    
                    String autoPopulateOLIName = '';
                    
                    if (newOLI.Product__c=='SOP' || newOLI.Product__c=='Intelligence'){
                      autoPopulateOLIName = '' + ((o.Billable_Advertiser__r.Name!=null && o.Billable_Advertiser__r.Name!= '') ? o.Billable_Advertiser__r.Name : '') + ' - ' +  ((newOli.Platform__c!=null && newOli.Platform__c!= '') ? newOli.Platform__c : '') + ' - ' +  ((newOli.Tactics__c!=null && newOli.Tactics__c!= '') ? newOli.Tactics__c : '') + ' - ' +  ((newOli.Unified_Fee__c!=null) ? String.valueOf(newOli.Unified_Fee__c) : '');
            
                    }else{
                      autoPopulateOLIName = '' + ((newOli.Strategy_Goal__c!=null && newOli.Strategy_Goal__c!= '') ? newOli.Strategy_Goal__c : '');// + ' - ' +  ((newOli.Client_Reference_ID__c!=null && newOli.Client_Reference_ID__c!= '') ? newOli.Client_Reference_ID__c : '') ;
                
                    }
                    if (autoPopulateOLIName.length()>80)  autoPopulateOLIName = autoPopulateOLIName.substring(0,80);
                    if (newOli.Name==null || newOli.Name=='' || newOli.Name=='Order Line Item Name'){
                      newOli.Name = autoPopulateOLIName;
                      //newOli.Strategy_Goal__c = autoPopulateOLIName;
                    
                    }"
                    
  Trigger 26
  
  "/*if (o.Start_Date__c!=null)
                        if (newOli.Start_Date__c==null) newOli.Start_Date__c = o.Start_Date__c;
                    
                    if (o.End_Date__c!=null)
                        if (newOli.End_Date__c==null)   newOli.End_Date__c = o.End_Date__c;*/
                }
            }
            /*if (newOLI.Product__c!=null && (newOLI.Product__c=='SOP' || newOLI.Product__c=='Intelligence')){
              if (newOli.Start_Date__c!=null && newOli.Start_Date__c <= Date.today()){
                  newOli.Status__c = 'Active';
                }
            }*/
        }
    }"
    
    Trigger 27
    
    "    private void processNetsuiteFields(Map<ID, Order_Line_Item__c> oldMap, List<Order_Line_Item__c> newMap){

      for (Order_Line_Item__c newOLI : newMap){
         if ( newOLI.NetSuite_Id__c!=null && newOLI.NetSuite_Id__c!='' && newOLI.Push_to_NetSuite__c==false){
            Order_Line_Item__c oldOLI = oldMap.get(newOLI.Id);
            
            if (oldOLI.name                != newOLI.name ||
          oldOLI.Strategy_Goal__c          != newOLI.Strategy_Goal__c ||
          oldOLI.Start_Date__c          != newOLI.Start_Date__c ||
          oldOLI.End_Date__c            != newOLI.End_Date__c ||
          oldOLI.Budget__c            != newOLI.Budget__c ||
          oldOLI.Unified_Fee__c          != newOLI.Unified_Fee__c ||
          oldOLI.Total_Advertiser_Cost__c      != newOLI.Total_Advertiser_Cost__c ||
          oldOLI.Platform__c            != newOLI.Platform__c ||
          oldOLI.Product__c            != newOLI.Product__c ||
          oldOLI.Publisher_Direct_Bill__c      != newOLI.Publisher_Direct_Bill__c ||
          oldOLI.Make_Good__c            != newOLI.Make_Good__c ||
          oldOLI.Added_Value__c          != newOLI.Added_Value__c ||
          oldOLI.Billing_Rate__c          != newOLI.Billing_Rate__c ||
          oldOLI.Billing_Rate_Type__c        != newOLI.Billing_Rate_Type__c ||
          oldOLI.SOP2_Initiative_Line_Item_ID__c  != newOLI.SOP2_Initiative_Line_Item_ID__c ||
          oldOLI.Client_Reference_ID__c      != newOLI.Client_Reference_ID__c ||
          oldOLI.Client_Fee_Line_Item_ID__c    != newOLI.Client_Fee_Line_Item_ID__c ||
          oldOLI.UP_Ad_Spend__c          != newOLI.UP_Ad_Spend__c ||
          oldOLI.Billable_Unified_Fee__c      != newOLI.Billable_Unified_Fee__c ||
          oldOLI.External_ID_1__c          != newOLI.External_ID_1__c ||
          oldOLI.External_ID_2__c          != newOLI.External_ID_2__c ||
          oldOLI.Status__c            != newOLI.Status__c )
          
          newOLI.Push_to_NetSuite__c = true;
        
          }
      }
"

Trigger 30

"""private void processDSOEmail(Map<ID, Order_Line_Item__c> newOrderMap){
    //2. If Account.Publisher_Notifications__c = True,
      //When an order line item.Status__c = Live and the orderline.Platform__c = Facebook,
      //if the Facebook DSO rep is on the billable account or brand, kick off the below email notification.
      //Send_Email_To_FB_DSO__c

      //3. If Publisher Notifications = True,
    //When an order line item is marked as Live and the Platform is Twitter,
    //if the Twitter DSO rep is on the billable account or brand, kick off the below email notification.
      //Send_Email_to_Twitter_DSO__c
""                                                                                                                                                                                                

/*Map<Id, Order_Line_Item__c> orderIdOltMap = new Map<Id, Order_Line_Item__c>();
    Set<Id> oliIdSet = new Set<Id>();
        for(Order_Line_Item__c newOli : newOrderMap.values()){

            if(newOli.Status__c == 'Live' && (newOli.Platform__c == 'Facebook' || newOli.Platform__c == 'Twitter' || newOli.Platform__c == 'LinkedIn')){
                if ((newOli.Platform__c == 'Facebook' && newOli.Send_Email_To_FB_DSO__c == false) ||
                  (newOli.Platform__c == 'LinkedIn' && newOli.Send_Email_To_LinkedIn_DSO__c == false) ||
                    (newOli.Platform__c == 'Twitter' && newOli.Send_Email_to_Twitter_DSO__c == false)){
                  orderIdOltMap.put(newOli.Order__c, newOli);
                  oliIdSet.add(newOli.Id);
              }
            }
        }

        if (oliIdSet.size() > 0){
          List<Order_Line_Item__c> oliList = [select Id, Order__c, Order__r.Opportunity__c, Order__r.Opportunity__r.AccountId,
                            Order__r.Opportunity__r.Brand__c, Platform__c,
                            Order__r.Opportunity__r.Account.Order_Publisher_Notification__c,
                            Billable_Account_FB_DSO__c, Billable_Account_Twitter_DSO__c,
                            Billable_Account_FB_DSO_2__c, Billable_Account_Twitter_DSO_2__c,
                            Billable_Account_FB_DSO_3__c, Billable_Account_Twitter_DSO_3__c,
                            Billable_Account_FB_DSO_4__c, Billable_Account_Twitter_DSO_4__c,
                            Billable_Account_FB_DSO_5__c, Billable_Account_Twitter_DSO_5__c,
                            Brand_FB_DSO__c, Brand_Twitter_DSO__c,
                            Brand_FB_DSO_2__c, Brand_Twitter_DSO_2__c,
                            Brand_FB_DSO_3__c, Brand_Twitter_DSO_3__c,
                            Brand_FB_DSO_4__c, Brand_Twitter_DSO_4__c,
                            Brand_FB_DSO_5__c, Brand_Twitter_DSO_5__c,
                            Send_Email_To_FB_DSO__c, Send_Email_to_Twitter_DSO__c,
                            Order_AM_Email__c, Order__r.Account_Manager__c,
                            Order_CM_Email__c, Order__r.Campaign_Manager__c,
                            Order_SC_Email__c, Order__r.Sales_Contact__c
                            from Order_Line_Item__c where Id IN :oliIdSet];

          Map<Id,Order_Line_Item__c> oliMap = new Map<Id,Order_Line_Item__c>();"
          
          Trigger 31
          
          "Map<Id,Id> oliIdAcctIdMap   = new Map<Id,Id>();
          Map<Id,Id> oliIdBrandIdMap   = new Map<Id,Id>();
          Set<Id> userIdSet = new Set<Id>(); // for storage of AM, CM, SC user ID

          Set<Id> oliAcctIdSet = new Set<Id>();

          for (Order_Line_Item__c o : oliList){

            if (o.Order__r.Opportunity__r.Account.Order_Publisher_Notification__c){
                oliMap.put(o.Id, o);
                /*if (o.Order__r.Opportunity__r.AccountId != null){
                    oliIdAcctIdMap.put(o.Id, o.Order__r.Opportunity__r.AccountId);
                    oliAcctIdSet.add(o.Order__r.Opportunity__r.AccountId);
                }*
                if (o.Order__r.Opportunity__r.Brand__c != null){
                    oliIdBrandIdMap.put(o.Id, o.Order__r.Opportunity__r.Brand__c);
                    oliAcctIdSet.add(o.Order__r.Opportunity__r.Brand__c);
                }
                if (o.Order__r.Account_Manager__c != null){
                    userIdSet.add(o.Order__r.Account_Manager__c);
                }
                if (o.Order__r.Campaign_Manager__c != null){
                    userIdSet.add(o.Order__r.Campaign_Manager__c);
                }
                if (o.Order__r.Sales_Contact__c != null){
                    userIdSet.add(o.Order__r.Sales_Contact__c);
                }
            }
          }
          
          Map<Id,String> userIdEmailMap = new Map<Id,String>();
          if (userIdSet.size() > 0){
            for (User u : [select Id, Email from User where ID IN :userIdSet limit 1000]){
                userIdEmailMap.put(u.Id,u.Email);
            }
          }"
          
  Trigger 38
  
  "/
  }

  private void createConfirmBusinessLocCase(Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems){
      
      
      Set<Id> oliIdSet = new Set<Id>();
      for(Integer i = 0 ; i < Trigger.new.size(); i++){
          
          if (    updatedOrderLineItems[i].Targeting_using_Business_Locations__c!=oldOrderLineItems[i].Targeting_using_Business_Locations__c &&
              updatedOrderLineItems[i].Targeting_using_Business_Locations__c==true){
                oliIdSet.add(updatedOrderLineItems[i].Id);
          }
        }
        
        Set<Id> oliWithConfirmBusinessLocCaseIdSet = new Set<Id>();
        if (oliIdSet.size()>0){
          for (Case c : [select Id, Order_Line_Item__c from Case 
                  where Order_Line_Item__c IN : oliIdSet
                  AND Area__c = 'Campaign Management'
                  AND Sub_Area__c = 'Confirm Business Locations']){
            oliWithConfirmBusinessLocCaseIdSet.add(c.Order_Line_Item__c);
          }
        
          List<Case> caseToInsertList = new List<Case>();
          for(Integer i = 0 ; i < Trigger.new.size(); i++){
            if (oliIdSet.contains(updatedOrderLineItems[i].Id) &&
              !oliWithConfirmBusinessLocCaseIdSet.contains(updatedOrderLineItems[i].Id)){
                
              caseToInsertList.add(  new Case( Order_Line_Item__c = updatedOrderLineItems[i].Id, 
                                Order__c = updatedOrderLineItems[i].Order__c, 
                                RecordTypeId = Constants.CAMPAIGN_MANAGEMENT_CASE_REC_TYPE,
                                    Area__c = 'Campaign Management',
                                    Sub_Area__c = 'Confirm Business Locations',
                                    Due_Date__c = date.today(),
                                    Subject = 'Please QA Business Locations',
                                    Description = 'This campaign is targeting using business locations. Please confirm: # of stores being targeted, # of locations per state, Targeted Business Locations ID'
              
                        ));
            }
          }
        
          if (caseToInsertList.size()>0)  insert caseToInsertList;
        }
  }"
  
  Trigger 39
  
  "private void createCase(Order_Line_Item__c[] oldOrderLineItems, Order_Line_Item__c[] updatedOrderLineItems, Map<ID, Order_Line_Item__c> oldMap){
    
        Map<Id, List<Order_Line_Item__c>> orderIdOliMap = new Map<Id, List<Order_Line_Item__c>>();
        Map<Id, Order_Line_Item__c> oliIdOldMap = new Map<Id, Order_Line_Item__c>();
        
        for(Integer i = 0 ; i < Trigger.new.size(); i++){
          
          if ( 
             (// if any of the below fields are updated regardless of the platform, create Update UP Line Item Case
              (  oldOrderLineItems[i].Strategy_Goal__c != updatedOrderLineItems[i].Strategy_Goal__c ||
                (oldOrderLineItems[i].Billing_Quantity__c != null && oldOrderLineItems[i].Billing_Quantity__c != updatedOrderLineItems[i].Billing_Quantity__c) ||
                (oldOrderLineItems[i].Billing_Rate__c != null && oldOrderLineItems[i].Billing_Rate__c != updatedOrderLineItems[i].Billing_Rate__c) ||
                (oldOrderLineItems[i].Billing_Rate_Type__c != null && oldOrderLineItems[i].Billing_Rate_Type__c != updatedOrderLineItems[i].Billing_Rate_Type__c) ||
                (oldOrderLineItems[i].Start_Date__c != null && oldOrderLineItems[i].Start_Date__c != updatedOrderLineItems[i].Start_Date__c) ||
                (oldOrderLineItems[i].End_Date__c != null && oldOrderLineItems[i].End_Date__c != updatedOrderLineItems[i].End_Date__c) ||
                // BOSP-536
                (oldOrderLineItems[i].Optimize_To__c != null && oldOrderLineItems[i].Optimize_To__c != updatedOrderLineItems[i].Optimize_To__c) ||
                (oldOrderLineItems[i].Publisher_Ad_Account__c != null && oldOrderLineItems[i].Publisher_Ad_Account__c != updatedOrderLineItems[i].Publisher_Ad_Account__c) ||
                // BOSP-554
                (oldOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c != null && oldOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c != updatedOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c) ||
                (oldOrderLineItems[i].Status__c != null && oldOrderLineItems[i].Status__c != updatedOrderLineItems[i].Status__c)
              )
              && // BOSP-554
           (
            (
             (updatedOrderLineItems[i].Strategy_Goal__c != null && updatedOrderLineItems[i].Strategy_Goal__c != '') &&
               (updatedOrderLineItems[i].Billing_Quantity__c != null && updatedOrderLineItems[i].Billing_Quantity__c > 0) &&
               (updatedOrderLineItems[i].Billing_Rate__c != null && updatedOrderLineItems[i].Billing_Rate__c > 0) &&
               (updatedOrderLineItems[i].Billing_Rate_Type__c != null && updatedOrderLineItems[i].Billing_Rate_Type__c != '') &&
               (updatedOrderLineItems[i].Start_Date__c != null) &&
               (updatedOrderLineItems[i].End_Date__c != null) &&
               (updatedOrderLineItems[i].Optimize_To__c != null && updatedOrderLineItems[i].Optimize_To__c != '') &&
               (updatedOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c != null && updatedOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c != '') &&
               (updatedOrderLineItems[i].Status__c != null && updatedOrderLineItems[i].Status__c != '')
            )
            ||
            (
             (updatedOrderLineItems[i].Strategy_Goal__c != null && updatedOrderLineItems[i].Strategy_Goal__c != '') &&
               (updatedOrderLineItems[i].Billing_Quantity__c != null && updatedOrderLineItems[i].Billing_Quantity__c > 0) &&
               (updatedOrderLineItems[i].Billing_Rate__c != null && updatedOrderLineItems[i].Billing_Rate__c > 0) &&
               (updatedOrderLineItems[i].Billing_Rate_Type__c != null && updatedOrderLineItems[i].Billing_Rate_Type__c != '') &&
               (updatedOrderLineItems[i].Start_Date__c != null) &&
               (updatedOrderLineItems[i].End_Date__c != null) &&
               (updatedOrderLineItems[i].Optimize_To__c != null && updatedOrderLineItems[i].Optimize_To__c != '') &&
               (updatedOrderLineItems[i].Publisher_Ad_Account__c != null) &&
               (!updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Snapchat')) &&
               (!updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Pinterest')) &&
               (!updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('LinkedIn')) &&
               (updatedOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c != null && updatedOrderLineItems[i].SOP2_Initiative_Line_Item_ID__c != '') &&
               (updatedOrderLineItems[i].Status__c != null && updatedOrderLineItems[i].Status__c != '')
            )
           )
             ) ||
             // if Platform__c = Facebook or Instagram AND any of the below fields are updated, create Update UP Line Item Case
             ( updatedOrderLineItems[i].Platform__c!=null && (updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Facebook') || updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Instagram')) &&
               (
                 (oldOrderLineItems[i].Age_Range_Low__c != null && oldOrderLineItems[i].Age_Range_Low__c != updatedOrderLineItems[i].Age_Range_Low__c) ||
                 (oldOrderLineItems[i].Age_Range_High__c != null && oldOrderLineItems[i].Age_Range_High__c != updatedOrderLineItems[i].Age_Range_High__c) ||
                 (oldOrderLineItems[i].Gender__c != null && oldOrderLineItems[i].Gender__c != updatedOrderLineItems[i].Gender__c) ||
                 (oldOrderLineItems[i].Device_Platform__c != null && oldOrderLineItems[i].Device_Platform__c != updatedOrderLineItems[i].Device_Platform__c) ||
                 (oldOrderLineItems[i].Publisher_Platform__c != null && oldOrderLineItems[i].Publisher_Platform__c != updatedOrderLineItems[i].Publisher_Platform__c) ||
                 (oldOrderLineItems[i].Facebook_Position__c != null && oldOrderLineItems[i].Facebook_Position__c != updatedOrderLineItems[i].Facebook_Position__c)
                 /* BOSP-536
                 oldOrderLineItems[i].Regions_States_Province__c != updatedOrderLineItems[i].Regions_States_Province__c ||
                 oldOrderLineItems[i].Cities__c != updatedOrderLineItems[i].Cities__c ||
                 oldOrderLineItems[i].Country_Facebook__c != updatedOrderLineItems[i].Country_Facebook__c ||
                 oldOrderLineItems[i].DMA__c != updatedOrderLineItems[i].DMA__c ||
                 oldOrderLineItems[i].Postcode__c != updatedOrderLineItems[i].Postcode__c ||
                 oldOrderLineItems[i].Targeting_using_Business_Locations__c != updatedOrderLineItems[i].Targeting_using_Business_Locations__c
                 */
               )
               && // BOSP-554
             (
               (updatedOrderLineItems[i].Age_Range_Low__c != null && updatedOrderLineItems[i].Age_Range_Low__c != '') &&
                 (updatedOrderLineItems[i].Age_Range_High__c != null && updatedOrderLineItems[i].Age_Range_High__c != '') &&
                 (updatedOrderLineItems[i].Gender__c != null && updatedOrderLineItems[i].Gender__c != '') &&
                 (updatedOrderLineItems[i].Device_Platform__c != null && updatedOrderLineItems[i].Device_Platform__c != '') &&
                 (updatedOrderLineItems[i].Publisher_Platform__c != null && updatedOrderLineItems[i].Publisher_Platform__c != '') &&
                 (updatedOrderLineItems[i].Facebook_Position__c != null && updatedOrderLineItems[i].Facebook_Position__c != '')
             )
             ) ||
             // if Platform__c = Twitter AND any of the below fields are updated, create Update UP Line Item Case
             ( updatedOrderLineItems[i].Platform__c!=null && updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Twitter') &&
               (
                 (oldOrderLineItems[i].Gender_Twitter__c != null && oldOrderLineItems[i].Gender_Twitter__c != updatedOrderLineItems[i].Gender_Twitter__c) ||
                 (oldOrderLineItems[i].Twitter_Placement__c != null && oldOrderLineItems[i].Twitter_Placement__c != updatedOrderLineItems[i].Twitter_Placement__c) ||
                 (oldOrderLineItems[i].Twitter_Funding_Source__c != null && oldOrderLineItems[i].Twitter_Funding_Source__c != updatedOrderLineItems[i].Twitter_Funding_Source__c) ||
                 (oldOrderLineItems[i].Twitter_IO_URL__c != null && oldOrderLineItems[i].Twitter_IO_URL__c != updatedOrderLineItems[i].Twitter_IO_URL__c)
               )
               && // BOSP-554
             (
               (updatedOrderLineItems[i].Gender_Twitter__c != null && updatedOrderLineItems[i].Gender_Twitter__c != '') &&
                 (updatedOrderLineItems[i].Twitter_Placement__c != null && updatedOrderLineItems[i].Twitter_Placement__c != '') &&
                 (updatedOrderLineItems[i].Twitter_Funding_Source__c != null && updatedOrderLineItems[i].Twitter_Funding_Source__c != '') &&
                 (updatedOrderLineItems[i].Twitter_IO_URL__c != null && updatedOrderLineItems[i].Twitter_IO_URL__c != '')
             )
             ) ||
             // if Platform__c = LinkedIn AND any of the below fields are updated, create Update UP Line Item Case
             ( updatedOrderLineItems[i].Platform__c!=null && updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('LinkedIn') &&
               (
                 // BOSP-536
                 //(oldOrderLineItems[i].Twitter_IO_URL__c != null && oldOrderLineItems[i].Age_Range_Low_LinkedIn__c != updatedOrderLineItems[i].Age_Range_Low_LinkedIn__c) ||
                 //(oldOrderLineItems[i].Twitter_IO_URL__c != null && oldOrderLineItems[i].Age_Range_High_LinkedIn__c != updatedOrderLineItems[i].Age_Range_High_LinkedIn__c) ||
                 (oldOrderLineItems[i].Gender_LinkedIn__c != null && oldOrderLineItems[i].Gender_LinkedIn__c != updatedOrderLineItems[i].Gender_LinkedIn__c) ||
                 (oldOrderLineItems[i].LinkedIn_Business_Account__c != null && oldOrderLineItems[i].LinkedIn_Business_Account__c != updatedOrderLineItems[i].LinkedIn_Business_Account__c)
               )
               && // BOSP-554
             (
               (updatedOrderLineItems[i].Gender_LinkedIn__c != null && updatedOrderLineItems[i].Gender_LinkedIn__c != '') &&
                 (updatedOrderLineItems[i].LinkedIn_Business_Account__c != null && updatedOrderLineItems[i].LinkedIn_Business_Account__c != '')
             )
             ) ||
             // if Platform__c = Pinterest AND any of the below fields are updated, create Update UP Line Item Case
             ( updatedOrderLineItems[i].Platform__c!=null && updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Pinterest') &&
               (
                 (oldOrderLineItems[i].Gender_Pinterest__c != null && oldOrderLineItems[i].Gender_Pinterest__c != updatedOrderLineItems[i].Gender_Pinterest__c)
               )
               && // BOSP-554
             (
               (updatedOrderLineItems[i].Gender_Pinterest__c != null && updatedOrderLineItems[i].Gender_Pinterest__c != '')
             )
             ) ||
             // if Platform__c = Snapchat AND any of the below fields are updated, create Update UP Line Item Case
             ( updatedOrderLineItems[i].Platform__c!=null && updatedOrderLineItems[i].Platform__c.equalsIgnoreCase('Snapchat') &&
               (
                 (oldOrderLineItems[i].Age_Groups__c != null && oldOrderLineItems[i].Age_Groups__c != updatedOrderLineItems[i].Age_Groups__c) ||
                 (oldOrderLineItems[i].Gender_Snapchat__c != null && oldOrderLineItems[i].Gender_Snapchat__c != updatedOrderLineItems[i].Gender_Snapchat__c)
               )
               && // BOSP-554
             (
               (updatedOrderLineItems[i].Age_Groups__c != null && updatedOrderLineItems[i].Age_Groups__c != '') &&
               (updatedOrderLineItems[i].Gender_Snapchat__c != null && updatedOrderLineItems[i].Gender_Snapchat__c != '')
             )
             )
            )
          {
            // BOSP-554
            if (updatedOrderLineItems[i].Status__c != 'New' && 
              updatedOrderLineItems[i].Status__c != 'Delayed' && 
              updatedOrderLineItems[i].Status__c != 'Canceled' && 
              updatedOrderLineItems[i].Status__c != 'Setup'){
              if (orderIdOliMap.containsKey(updatedOrderLineItems[i].Order__c)){
                      orderIdOliMap.get(updatedOrderLineItems[i].Order__c).add(updatedOrderLineItems[i]);
                  }else{
                      orderIdOliMap.put(updatedOrderLineItems[i].Order__c, new List<Order_Line_Item__c>{updatedOrderLineItems[i]});
                  }
                  oliIdOldMap.put(oldOrderLineItems[i].Id, oldOrderLineItems[i]);
            }
          ""// for main if clause
        }
          
        StringUtils su = new StringUtils();
        List<Case> newCaseList = new List<Case>();
        
        for(Order__c order : [Select Id, Billable_Advertiser__r.Name, Brand__r.Name, Opportunity__c, Agency__r.Name, Opportunity__r.AccountId, Account_Manager__c, Campaign_Manager__c From Order__c Where Id IN :orderIdOliMap.keySet()]){
      if(orderIdOliMap.containsKey(order.Id)){
                for (Order_Line_Item__c oli : orderIdOliMap.get(order.Id)){
                
                  Order_Line_Item__c oldoli = oliIdOldMap.get(oli.Id);
          String subject = 'Campaign IO Change Occurred: '+
                                    su.subjectize(oli.Platform__c, 'dash') +
                                    su.subjectize(oli.Tactics__c, 'dash') +
                                    su.subjectize(oli.Sub_Tactics__c, 'dash') +
                                    su.subjectize(order.Billable_Advertiser__r.Name, 'comma') +
                                    su.subjectize(order.Brand__r.Name, 'comma') +
                                    su.subjectize(order.Agency__r.Name, 'dash') +
                                    su.dateSubjectize(oli.Start_Date__c);
                  
          Case c = new Case(Subject = subject, Order_Line_Item__c = oli.ID, Opportunity__c = order.Opportunity__c, Order__c = order.Id, AccountId = order.Opportunity__r.AccountId);
          c.RecordTypeId = OrderLineItemTriggerHandler.cmCaseRecTypeId; 
          c.Area__c = 'Campaign Management';
          c.Sub_Area__c = 'Update UP Line Item';
                  c.Status = 'New';
                    copyOliToCaseFields(c, oli);
                  c.Field_History__c = getChangedValue(oldoli, oli);
                  
                  newCaseList.add(c);"""
                  
Trigger 41

"Clone all Update UP Line Item cases BOSP-278 as reference
                  Case tempCase = c.clone(false, true, false, false);
            tempCase.Subject = 'QA: ' + c.Subject;
            tempCase.Description =   '- Budget adjusted in Publisher \n' +
                      '- Budget adjusted in UP Line Item \n' +
                      '- Budget adjusted in Optimizer (if on) \n' +
                      '- Flight adjusted in AdSet/Campaign (publisher) \n' +
                      '- Flight adjusted in UP Line Item \n' +
                      '- Flight adjusted in Optimizer (if on)';
            
            tempCase.QA_Date_Time__c = null; // clear the QA date time to avoid multiple cases created
            tempCase.Take_Down_Date_Time__c = null; // clear the take down time to avoid multiple cases created
            newCaseList.add(tempCase);
              }
            }
        }


        if(newCaseList.size() > 0) insert newCaseList;
        //if(oliList.size() > 0) update oliList;"
        
        Trigger 42
        
        "public Case copyOliToCaseFields(Case c, Order_Line_Item__c olt){
    
    if (olt.LinkedIn_Business_Account__c!=null) c.LinkedIn_Business_Account__c = olt.LinkedIn_Business_Account__c;
    if (olt.Planner__c!=null) c.Planner__c = olt.Planner__c;
    if (olt.SOP_INTEGRATION_ID__C!=null) c.SOP_INTEGRATION_ID__C = olt.SOP_INTEGRATION_ID__C;
    if (olt.SOP_CAMPAIGN_ID__C!=null) c.SOP_CAMPAIGN_ID__C = olt.SOP_CAMPAIGN_ID__C;
    if (olt.BILLING_QUANTITY__C!=null) c.BILLING_QUANTITY__C = olt.BILLING_QUANTITY__C;
    if (olt.BILLING_RATE__C!=null) c.BILLING_RATE__C = olt.BILLING_RATE__C;
    if (olt.BILLING_RATE_TYPE__C!=null) c.BILLING_RATE_TYPE__C = olt.BILLING_RATE_TYPE__C;
    if (olt.START_DATE__C!=null) c.ORDER_LINE_ITEM_START_DATE__C = olt.START_DATE__C;
    if (olt.BUDGET__C!=null) c.Contracted_Media_Cost__c = olt.BUDGET__C;
    if (olt.SOP_UPDATE_DATE__C!=null) c.SOP_UPDATE_DATE__C = olt.SOP_UPDATE_DATE__C;
    if (olt.PLATFORM__C!=null) c.PLATFORM__C = olt.PLATFORM__C;
    if (olt.SUB_TACTICS__C!=null) c.SUB_TACTICS__C = olt.SUB_TACTICS__C;
    if (olt.TACTICS__C!=null) c.TACTICS__C = olt.TACTICS__C;
    if (olt.TARGETING_GROUP__C!=null) c.TARGETING_GROUP__C = olt.TARGETING_GROUP__C;
    if (olt.CUSTOM_TARGETING_GROUP__C!=null) c.CUSTOM_TARGETING_GROUP__C = olt.CUSTOM_TARGETING_GROUP__C;
    
    if (olt.Country_Facebook__c!=null)  c.Country_Facebook__c = olt.Country_Facebook__c;
    if (olt.Country_Twitter__c!=null)   c.Country_Twitter__c = olt.Country_Twitter__c;
    if (olt.Country_StumbleUpon__c!=null) c.Country_StumbleUpon__c = olt.Country_StumbleUpon__c;
    if (olt.Country_LinkedIn__c!=null)  c.Country_LinkedIn__c = olt.Country_LinkedIn__c;
    if (olt.Country_YouTube__c!=null)   c.Country_YouTube__c = olt.Country_YouTube__c;
    if (olt.Country_Disqus__c!=null)    c.Country_Disqus__c = olt.Country_Disqus__c;
    
    
    if (olt.PUBLISHER_DIRECT_BILL_TO_ADVERTISER__C!=null) c.PUBLISHER_DIRECT_BILL_TO_ADVERTISER__C = olt.PUBLISHER_DIRECT_BILL_TO_ADVERTISER__C;
    if (olt.SOP_CAMPAIGN_URL__C!=null) c.SOP_CAMPAIGN_URL__C = olt.SOP_CAMPAIGN_URL__C;
    if (olt.SOP_CAMPAIGN_NAME__C!=null) c.SOP_CAMPAIGN_NAME__C = olt.SOP_CAMPAIGN_NAME__C;
    if (olt.SOP_EXTERNAL_INTEGRATION_ID__C!=null) c.SOP_EXTERNAL_INTEGRATION_ID__C = olt.SOP_EXTERNAL_INTEGRATION_ID__C;
    if (olt.SOP_MEDIA_COST__C!=null) c.SOP_MEDIA_COST__C = olt.SOP_MEDIA_COST__C;
    if (olt.SOP_SPEND_TYPE__C!=null) c.SOP_SPEND_TYPE__C = olt.SOP_SPEND_TYPE__C;
    if (olt.SOP_SPEND_RATE__C!=null) c.SOP_SPEND_RATE__C = olt.SOP_SPEND_RATE__C;
    if (olt.SOP_GOAL__C!=null) c.SOP_GOAL__C = olt.SOP_GOAL__C;
    if (olt.SOP_PASS_THROUGH__C!=null) c.SOP_PASS_THROUGH__C = olt.SOP_PASS_THROUGH__C;
    if (olt.SOP_CONVERSION_GOAL__C!=null) c.SOP_CONVERSION_GOAL__C = olt.SOP_CONVERSION_GOAL__C;
    if (olt.SOP_IMPRESSIONS__C!=null) c.SOP_IMPRESSIONS__C = olt.SOP_IMPRESSIONS__C;
    if (olt.SOP_CLICKS__C!=null) c.SOP_CLICKS__C = olt.SOP_CLICKS__C;
    if (olt.SOP_CONVERSIONS__C!=null) c.SOP_CONVERSIONS__C = olt.SOP_CONVERSIONS__C;
    if (olt.UNIFIED_FEE__C!=null) c.Contracted_Unified_Fee__c = olt.UNIFIED_FEE__C;
    if (olt.BILLABLE_UNIFIED_FEE__C!=null) c.BILLABLE_UNIFIED_FEE__C = olt.BILLABLE_UNIFIED_FEE__C;
    if (olt.BILLABLE_TOTAL_ADVERTISER_COST__C!=null) c.Billable_Fixed_Monthly_Advertiser_Cost__c = olt.BILLABLE_TOTAL_ADVERTISER_COST__C;
    if (olt.CPA_ADVERTISER_COST__C!=null) c.CPA_ADVERTISER_COST__C = olt.CPA_ADVERTISER_COST__C;
    if (olt.BILLABLE_MEDIA_COST__C!=null) c.BILLABLE_MEDIA_COST__C = olt.BILLABLE_MEDIA_COST__C;
    if (olt.OF_BUDGET_COST_PLUS_ADVER__C!=null) c.OF_BUDGET_COST_PLUS_ADVER__C = olt.OF_BUDGET_COST_PLUS_ADVER__C;
    if (olt.COST_PLUS__C!=null) c.COST_PLUS__C = olt.COST_PLUS__C;
    if (olt.UNIFIED_FEE_OVERAGE__C!=null) c.UNIFIED_FEE_OVERAGE__C = olt.UNIFIED_FEE_OVERAGE__C;
    if (olt.CLIENT_REFERENCE_ID__C!=null) c.CLIENT_REFERENCE_ID__C = olt.CLIENT_REFERENCE_ID__C;
    if (olt.AGE_RANGE_HIGH__C!=null) c.AGE_RANGE_HIGH__C = olt.AGE_RANGE_HIGH__C;
    if (olt.AGE_RANGE_LOW__C!=null) c.AGE_RANGE_LOW__C = olt.AGE_RANGE_LOW__C;
    if (olt.GENDER__C!=null) c.GENDER__C = olt.GENDER__C;
    if (olt.REGIONS_STATES_PROVINCE__C!=null) c.REGIONS_STATES_PROVINCE__C = olt.REGIONS_STATES_PROVINCE__C;
    if (olt.CITIES__C!=null) c.CITIES__C = olt.CITIES__C;
    if (olt.DMA__C!=null) c.DMA__C = olt.DMA__C;
    if (olt.POSTCODE__C!=null) c.POSTCODE__C = olt.POSTCODE__C;
    if (olt.APPROXIMATE_ETHNICITY_TARGETING__C!=null) c.APPROXIMATE_ETHNICITY_TARGETING__C = olt.APPROXIMATE_ETHNICITY_TARGETING__C;
    if (olt.PLACEMENT__C!=null) c.PLACEMENT__C = olt.PLACEMENT__C;
    if (olt.DEVICE_TARGETING__C!=null) c.DEVICE_TARGETING__C = olt.DEVICE_TARGETING__C;
    if (olt.SOCIAL_TARGETING__C!=null) c.SOCIAL_TARGETING__C = olt.SOCIAL_TARGETING__C;
    if (olt.PRECISE_INTERESTS__C!=null) c.PRECISE_INTERESTS__C = olt.PRECISE_INTERESTS__C;
    if (olt.NEW_LINE_BROAD_INTEREST__C!=null) c.NEW_LINE_BROAD_INTEREST__C = olt.NEW_LINE_BROAD_INTEREST__C;
    if (olt.BEHAVIORAL_3RD_PARTY__C!=null) c.BEHAVIORAL_3RD_PARTY__C = olt.BEHAVIORAL_3RD_PARTY__C;
    if (olt.INCLUDE_EXCLUDE_EXSISTING_FAN__C!=null) c.INCLUDE_EXCLUDE_EXSISTING_FAN__C = olt.INCLUDE_EXCLUDE_EXSISTING_FAN__C;
    if (olt.EXISTING_CRM_LIST_CUSTOMERS__C!=null) c.EXISTING_CRM_LIST_CUSTOMERS__C = olt.EXISTING_CRM_LIST_CUSTOMERS__C;
    if (olt.LOOKALIKE_S_OF_CRM_LIST_CUSTOMERS__C!=null) c.LOOKALIKE_S_OF_CRM_LIST_CUSTOMERS__C = olt.LOOKALIKE_S_OF_CRM_LIST_CUSTOMERS__C;
    if (olt.COOKIEDATA_APPLIED_TO_EXCHANGE_INVENTORY__C!=null) c.COOKIEDATA_APPLIED_TO_EXCHANGE_INVENTORY__C = olt.COOKIEDATA_APPLIED_TO_EXCHANGE_INVENTORY__C;
    if (olt.GENDER_TWITTER__C!=null) c.GENDER_TWITTER__C = olt.GENDER_TWITTER__C;
    if (olt.REGIONS_STATES_PROVINCE_TWITTER__C!=null) c.REGIONS_STATES_PROVINCE_TWITTER__C = olt.REGIONS_STATES_PROVINCE_TWITTER__C;
    if (olt.CITIES_TWITTER__C!=null) c.CITIES_TWITTER__C = olt.CITIES_TWITTER__C;
    if (olt.INTEREST_KEYWORDS__C!=null) c.INTEREST_KEYWORDS__C = olt.INTEREST_KEYWORDS__C;
    if (olt.DEVICE_TARGETING_TWITTER__C!=null) c.DEVICE_TARGETING_TWITTER__C = olt.DEVICE_TARGETING_TWITTER__C;
    if (olt.INCLUDE_OR_EXCLUDE_EXISTING_FOLLOWERS__C!=null) c.INCLUDE_OR_EXCLUDE_EXISTING_FOLLOWERS__C = olt.INCLUDE_OR_EXCLUDE_EXISTING_FOLLOWERS__C;
    if (olt.HANDLE_LOOK_A_LIKE__C!=null) c.HANDLE_LOOK_A_LIKE__C = olt.HANDLE_LOOK_A_LIKE__C;
    if (olt.KEYWORD_TIMELINE_TARGETING__C!=null) c.KEYWORD_TIMELINE_TARGETING__C = olt.KEYWORD_TIMELINE_TARGETING__C;
    if (olt.AGE_RANGE_LOW_STUMBLEUPON__C!=null) c.AGE_RANGE_LOW_STUMBLEUPON__C = olt.AGE_RANGE_LOW_STUMBLEUPON__C;
    if (olt.AGE_RANGE_HIGH_STUMBLEUPON__C!=null) c.AGE_RANGE_HIGH_STUMBLEUPON__C = olt.AGE_RANGE_HIGH_STUMBLEUPON__C;
    if (olt.GENDER_STUMBLEUPON__C!=null) c.GENDER_STUMBLEUPON__C = olt.GENDER_STUMBLEUPON__C;
    if (olt.REGIONS_STATES_PROVINCE_STUMBLEUPON__C!=null) c.REGIONS_STATES_PROVINCE_STUMBLEUPON__C = olt.REGIONS_STATES_PROVINCE_STUMBLEUPON__C;
    if (olt.CITIES_STUMBLEUPON__C!=null) c.CITIES_STUMBLEUPON__C = olt.CITIES_STUMBLEUPON__C;
    if (olt.DMA_STUMBLEUPON__C!=null) c.DMA_STUMBLEUPON__C = olt.DMA_STUMBLEUPON__C;
    if (olt.INTEREST_TARGETING_STUMBLEUPON__C!=null) c.INTEREST_TARGETING_STUMBLEUPON__C = olt.INTEREST_TARGETING_STUMBLEUPON__C;
    if (olt.DEVICE_TARGETING_STUMBLEUPON__C!=null) c.DEVICE_TARGETING_STUMBLEUPON__C = olt.DEVICE_TARGETING_STUMBLEUPON__C;
    if (olt.AGE_RANGE_LOW_LINKEDIN__C!=null) c.AGE_RANGE_LOW_LINKEDIN__C = olt.AGE_RANGE_LOW_LINKEDIN__C;
    if (olt.AGE_RANGE_HIGH_LINKEDIN__C!=null) c.AGE_RANGE_HIGH_LINKEDIN__C = olt.AGE_RANGE_HIGH_LINKEDIN__C;
    if (olt.GENDER_LINKEDIN__C!=null) c.GENDER_LINKEDIN__C = olt.GENDER_LINKEDIN__C;
    if (olt.REGIONS_STATES_PROVINCE_LINKEDIN__C!=null) c.REGIONS_STATES_PROVINCE_LINKEDIN__C = olt.REGIONS_STATES_PROVINCE_LINKEDIN__C;
    if (olt.CITIES_LINKEDIN__C!=null) c.CITIES_LINKEDIN__C = olt.CITIES_LINKEDIN__C;
    if (olt.DMA_LINKEDIN__C!=null) c.DMA_LINKEDIN__C = olt.DMA_LINKEDIN__C;
    if (olt.COUNTRIES__C!=null) c.COUNTRIES__C = olt.COUNTRIES__C;
    if (olt.GENERAL_TARGETING_DESCRIPTION__C!=null) c.GENERAL_TARGETING_DESCRIPTION__C = olt.GENERAL_TARGETING_DESCRIPTION__C;
    if (olt.COMPANY_CATEGORIES__C!=null) c.COMPANY_CATEGORIES__C = olt.COMPANY_CATEGORIES__C;
    if (olt.COMPANY_SIZE_LINKEDIN__C!=null) c.COMPANY_SIZE_LINKEDIN__C = olt.COMPANY_SIZE_LINKEDIN__C;
    if (olt.COMPANY_SIZE__C!=null) c.COMPANY_SIZE__C = olt.COMPANY_SIZE__C;
    if (olt.JOB_TITLE__C!=null) c.JOB_TITLE__C = olt.JOB_TITLE__C;
    if (olt.JOB_FUNCTION__C!=null) c.JOB_FUNCTION__C = olt.JOB_FUNCTION__C;
    if (olt.JOB_CATEGORIES__C!=null) c.JOB_CATEGORIES__C = olt.JOB_CATEGORIES__C;
    if (olt.AGE_RANGE_LOW_YOUTUBE__C!=null) c.AGE_RANGE_LOW_YOUTUBE__C = olt.AGE_RANGE_LOW_YOUTUBE__C;
    if (olt.AGE_RANGE_HIGH_YOUTUBE__C!=null) c.AGE_RANGE_HIGH_YOUTUBE__C = olt.AGE_RANGE_HIGH_YOUTUBE__C;
    if (olt.GENDER_YOUTUBE__C!=null) c.GENDER_YOUTUBE__C = olt.GENDER_YOUTUBE__C;
    if (olt.REGIONS_STATES_PROVINCE_YOUTUBE__C!=null) c.REGIONS_STATES_PROVINCE_YOUTUBE__C = olt.REGIONS_STATES_PROVINCE_YOUTUBE__C;
    if (olt.CITIES_YOUTTUBE__C!=null) c.CITIES_YOUTTUBE__C = olt.CITIES_YOUTTUBE__C;
    if (olt.CONTEXTUAL_TARGETING__C!=null) c.CONTEXTUAL_TARGETING__C = olt.CONTEXTUAL_TARGETING__C;
    if (olt.DEVICE_TARGETING_YOUTUBE__C!=null) c.DEVICE_TARGETING_YOUTUBE__C = olt.DEVICE_TARGETING_YOUTUBE__C;
    if (olt.REGIONS_STATES_PROVINCE_DISQUS__C!=null) c.REGIONS_STATES_PROVINCE_DISQUS__C = olt.REGIONS_STATES_PROVINCE_DISQUS__C;
    if (olt.CITIES_DISQUS__C!=null) c.CITIES_DISQUS__C = olt.CITIES_DISQUS__C;
    if (olt.DMA_DISQUS__C!=null) c.DMA_DISQUS__C = olt.DMA_DISQUS__C;
    if (olt.DEVICE__C!=null) c.DEVICE__C = olt.DEVICE__C;
    if (olt.TRACKING_TAGS_NOTES__C!=null) c.TRACKING_TAGS_NOTES__C = olt.TRACKING_TAGS_NOTES__C;
    if (olt.OPTIMIZE_TO__C!=null) c.OPTIMIZE_TO__C = olt.OPTIMIZE_TO__C;
    if (olt.PACING_NOTES__C!=null) c.PACING_NOTES__C = olt.PACING_NOTES__C;
    if (olt.TWITTER_HANDLE__C!=null) c.TWITTER_HANDLE__C = olt.TWITTER_HANDLE__C;
    if (olt.FB_PAGE__C!=null) c.FB_PAGE__C = olt.FB_PAGE__C;
    if (olt.FB_SPEND_ACCOUNT__C!=null) c.FB_SPEND_ACCOUNT__C = olt.FB_SPEND_ACCOUNT__C;
    if (olt.TRACKING_TAGS__C!=null) c.TRACKING_TAGS__C = olt.TRACKING_TAGS__C;
    if (olt.Name!=null) c.NAME__C = olt.Name;
    c.Line_Item_Status__c = 'New';
    if (olt.End_Date__c!=null) c.Order_Line_Item_End_Date__c = olt.End_Date__c;
    if (olt.Total_Advertiser_Cost__c!=null) c.Total_Contracted_Advertiser_Cost__c = olt.Total_Advertiser_Cost__c;
    if (olt.Case__c!=null) c.Proposal_RFP__c = olt.Case__c;
    if (olt.Estimated_Rate_Type__c!=null) c.Goal_Rate_Type__c = olt.Estimated_Rate_Type__c;
    if (olt.Estimated_Rate_Per_Unit__c!=null) c.Goal_Rate_Per_Unit__c = olt.Estimated_Rate_Per_Unit__c;
    if (olt.Estimated_Imp_Clicks_Actions__c!=null) c.Goal_Imp_Clicks_Actions__c = olt.Estimated_Imp_Clicks_Actions__c;
    if (olt.Over_Delivery__c!=null) c.Over_Under_Delivery__c = olt.Over_Delivery__c;
    if (olt.Strategy_Goal__c!=null) c.SOP_Campaign_Description__c = olt.Strategy_Goal__c;
    
    if (olt.NOTES__C!=null) c.ORDER_LINE_ITEM_NOTES__C = olt.NOTES__C;
    return c;
  }"
  
  
  Trigger 43
  
  "public String getChangedValue(Order_Line_Item__c oldOli, Order_Line_Item__c newOli){
    
    
    String type='Order_Line_Item__c';
  Map<String, Schema.SObjectType> schemaMap = Schema.getGlobalDescribe();
  Schema.SObjectType oliSchema = schemaMap.get(type);
  Map<String, Schema.SObjectField> fieldMap = oliSchema.getDescribe().fields.getMap();
  
  /*for (String fieldName: fieldMap.keySet()) {
    System.debug('##Field API Name='+fieldName);// list of all field API name
    fieldMap.get(fieldName).getDescribe().getLabel();//It provides to get the object fields label.
  }
""/
    
    String changesToNote = '';
    
    if (oldOli.Strategy_Goal__c != newOli.Strategy_Goal__c){
        changesToNote += fieldMap.get('Strategy_Goal__c').getDescribe().getLabel();
        changesToNote += ' changed value from ' + oldOli.Strategy_Goal__c + ' to ' + newOli.Strategy_Goal__c + '\n';
    }
    if (oldOli.BILLING_QUANTITY__C != newOli.BILLING_QUANTITY__C){
        changesToNote += fieldMap.get('Billing_Quantity__c').getDescribe().getLabel();  //'Billing Quantity';
        changesToNote += ' changed value from ' + oldOli.BILLING_QUANTITY__C + ' to ' + newOli.BILLING_QUANTITY__C + '\n';
    }
    if (oldOli.BILLING_RATE__C != newOli.BILLING_RATE__C){
        changesToNote += fieldMap.get('Billing_Rate__c').getDescribe().getLabel(); //'Billing Rate';
        changesToNote += ' changed value from ' + oldOli.BILLING_RATE__C + ' to ' + newOli.BILLING_RATE__C + '\n';
    }
    if (oldOli.BILLING_RATE_TYPE__C != newOli.BILLING_RATE_TYPE__C){
        changesToNote += fieldMap.get('Billing_Rate_Type__c').getDescribe().getLabel(); //'Billing Rate Type';
        changesToNote += ' changed value from ' + oldOli.BILLING_RATE_TYPE__C + ' to ' + newOli.BILLING_RATE_TYPE__C + '\n';
    }
    if (oldOli.Start_Date__c != newOli.Start_Date__c){
        changesToNote += fieldMap.get('Start_Date__c').getDescribe().getLabel(); //'Start Date';
        changesToNote += ' changed value from ' + oldOli.Start_Date__c.format() + ' to ' + newOli.Start_Date__c.format() + '\n';
    }
    if (oldOli.End_Date__c != newOli.End_Date__c){
        changesToNote += fieldMap.get('End_Date__c').getDescribe().getLabel(); //'End Date';
        changesToNote += ' changed value from ' + oldOli.End_Date__c.format() + ' to ' + newOli.End_Date__c.format() + '\n';
    }
    
    // Facebook
    if (oldOli.Age_Range_Low__c != newOli.Age_Range_Low__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Age_Range_Low__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Age_Range_Low__c + ' to ' + newOli.Age_Range_Low__c + '\n';
    }
    if (oldOli.Age_Range_High__c != newOli.Age_Range_High__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Age_Range_High__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Age_Range_High__c + ' to ' + newOli.Age_Range_High__c + '\n';
    }
    if (oldOli.Gender__c != newOli.Gender__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Gender__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Gender__c + ' to ' + newOli.Gender__c + '\n';
    }
    if (oldOli.Regions_States_Province__c != newOli.Regions_States_Province__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Regions_States_Province__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Regions_States_Province__c + ' to ' + newOli.Regions_States_Province__c + '\n';
    }
    if (oldOli.Cities__c != newOli.Cities__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Cities__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Cities__c + ' to ' + newOli.Cities__c + '\n';
    }
    if (oldOli.Country_Facebook__c != newOli.Country_Facebook__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Country_Facebook__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Country_Facebook__c + ' to ' + newOli.Country_Facebook__c + '\n';
    }
    if (oldOli.DMA__c != newOli.DMA__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('DMA__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.DMA__c + ' to ' + newOli.DMA__c + '\n';
    }
    if (oldOli.Postcode__c != newOli.Postcode__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Postcode__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Postcode__c + ' to ' + newOli.Postcode__c + '\n';
    }
    if (oldOli.Approximate_Ethnicity_Targeting__c != newOli.Approximate_Ethnicity_Targeting__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Approximate_Ethnicity_Targeting__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Approximate_Ethnicity_Targeting__c + ' to ' + newOli.Approximate_Ethnicity_Targeting__c + '\n';
    }
    if (oldOli.Placement__c != newOli.Placement__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Placement__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Placement__c + ' to ' + newOli.Placement__c + '\n';
    }
    if (oldOli.Device_Targeting__c != newOli.Device_Targeting__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Device_Targeting__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Device_Targeting__c + ' to ' + newOli.Device_Targeting__c + '\n';
    }
    if (oldOli.Social_Targeting__c != newOli.Social_Targeting__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Social_Targeting__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Social_Targeting__c + ' to ' + newOli.Social_Targeting__c + '\n';
    }
    if (oldOli.Precise_Interests__c != newOli.Precise_Interests__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Precise_Interests__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Precise_Interests__c + ' to ' + newOli.Precise_Interests__c + '\n';
    }
    if (oldOli.New_Line_Broad_Interest__c != newOli.New_Line_Broad_Interest__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('New_Line_Broad_Interest__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.New_Line_Broad_Interest__c + ' to ' + newOli.New_Line_Broad_Interest__c + '\n';
    }
    if (oldOli.Behavioral_3rd_Party__c != newOli.Behavioral_3rd_Party__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Behavioral_3rd_Party__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Behavioral_3rd_Party__c + ' to ' + newOli.Behavioral_3rd_Party__c + '\n';
    }
    if (oldOli.Existing_CRM_List_Customers__c != newOli.Existing_CRM_List_Customers__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Existing_CRM_List_Customers__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Existing_CRM_List_Customers__c + ' to ' + newOli.Existing_CRM_List_Customers__c + '\n';
    }
    if (oldOli.Lookalike_s_of_CRM_List_Customers__c != newOli.Lookalike_s_of_CRM_List_Customers__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Lookalike_s_of_CRM_List_Customers__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Lookalike_s_of_CRM_List_Customers__c + ' to ' + newOli.Lookalike_s_of_CRM_List_Customers__c + '\n';
    }
    if (oldOli.CookieData_Applied_to_Exchange_Inventory__c != newOli.CookieData_Applied_to_Exchange_Inventory__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('CookieData_Applied_to_Exchange_Inventory__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.CookieData_Applied_to_Exchange_Inventory__c + ' to ' + newOli.CookieData_Applied_to_Exchange_Inventory__c + '\n';
    }
    if (oldOli.Device_Platform__c != newOli.Device_Platform__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Device_Platform__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Device_Platform__c + ' to ' + newOli.Device_Platform__c + '\n';
    }
    if (oldOli.Publisher_Platform__c != newOli.Publisher_Platform__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Publisher_Platform__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Publisher_Platform__c + ' to ' + newOli.Publisher_Platform__c + '\n';
    }
    if (oldOli.Facebook_Position__c != newOli.Facebook_Position__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Facebook_Position__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Facebook_Position__c + ' to ' + newOli.Facebook_Position__c + '\n';
    }
    
    if (newOli.Platform__c.equalsIgnoreCase('Facebook') && oldOli.Targeting_using_Business_Locations__c != newOli.Targeting_using_Business_Locations__c){
      changesToNote +=  'Facebook ' + 
                fieldMap.get('Targeting_using_Business_Locations__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Targeting_using_Business_Locations__c + ' to ' + newOli.Targeting_using_Business_Locations__c + '\n';
    }
    """
    " // Instagram
    if (newOli.Platform__c.equalsIgnoreCase('Instagram') && oldOli.Targeting_using_Business_Locations__c != newOli.Targeting_using_Business_Locations__c){
      changesToNote +=  'Instagram ' + 
                fieldMap.get('Targeting_using_Business_Locations__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Targeting_using_Business_Locations__c + ' to ' + newOli.Targeting_using_Business_Locations__c + '\n';
    }"

" // Pinterest
    if (newOli.Platform__c.equalsIgnoreCase('Pinterest') && oldOli.Gender_Pinterest__c != newOli.Gender_Pinterest__c){
      changesToNote +=  'Pinterest ' + 
                fieldMap.get('Gender_Pinterest__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Gender_Pinterest__c + ' to ' + newOli.Gender_Pinterest__c + '\n';"

" // Snapchat
    if (newOli.Platform__c.equalsIgnoreCase('Snapchat') && oldOli.Age_Groups__c != newOli.Age_Groups__c){
      changesToNote +=  'Snapchat ' + 
                fieldMap.get('Age_Groups__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Age_Groups__c + ' to ' + newOli.Age_Groups__c + '\n';
    }
    if (newOli.Platform__c.equalsIgnoreCase('Snapchat') && oldOli.Gender_Snapchat__c != newOli.Gender_Snapchat__c){
      changesToNote +=  'Snapchat ' + 
                fieldMap.get('Gender_Snapchat__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Gender_Snapchat__c + ' to ' + newOli.Gender_Snapchat__c + '\n';"

"// Twitter
    if (oldOli.Gender_Twitter__c != newOli.Gender_Twitter__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Gender_Twitter__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Gender_Twitter__c + ' to ' + newOli.Gender_Twitter__c + '\n';
    }
    if (oldOli.Twitter_Placement__c != newOli.Twitter_Placement__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Twitter_Placement__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Twitter_Placement__c + ' to ' + newOli.Twitter_Placement__c + '\n';
    }
    if (oldOli.Regions_States_Province_Twitter__c != newOli.Regions_States_Province_Twitter__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Regions_States_Province_Twitter__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Regions_States_Province_Twitter__c + ' to ' + newOli.Regions_States_Province_Twitter__c + '\n';
    }
    if (oldOli.Cities_Twitter__c != newOli.Cities_Twitter__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Cities_Twitter__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Cities_Twitter__c + ' to ' + newOli.Cities_Twitter__c + '\n';
    }
    if (oldOli.Country_Twitter__c != newOli.Country_Twitter__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Country_Twitter__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Country_Twitter__c + ' to ' + newOli.Country_Twitter__c + '\n';
    }
    if (oldOli.Handle_Look_a_like__c != newOli.Handle_Look_a_like__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Handle_Look_a_like__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Handle_Look_a_like__c + ' to ' + newOli.Handle_Look_a_like__c + '\n';
    }
    if (oldOli.Interest_Keywords__c != newOli.Interest_Keywords__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Interest_Keywords__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Interest_Keywords__c + ' to ' + newOli.Interest_Keywords__c + '\n';
    }
    if (oldOli.Device_Targeting_Twitter__c != newOli.Device_Targeting_Twitter__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Device_Targeting_Twitter__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Device_Targeting_Twitter__c + ' to ' + newOli.Device_Targeting_Twitter__c + '\n';
    }
    if (oldOli.Include_or_Exclude_Existing_Followers__c != newOli.Include_or_Exclude_Existing_Followers__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Include_or_Exclude_Existing_Followers__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Include_or_Exclude_Existing_Followers__c + ' to ' + newOli.Include_or_Exclude_Existing_Followers__c + '\n';
    }
    if (oldOli.Keyword_Timeline_Targeting__c != newOli.Keyword_Timeline_Targeting__c){
      changesToNote +=  'Twitter ' + 
                fieldMap.get('Keyword_Timeline_Targeting__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Keyword_Timeline_Targeting__c + ' to ' + newOli.Keyword_Timeline_Targeting__c + '\n';
    }
    
    // LinkedIn
    if (oldOli.Age_Range_Low_Linkedin__c != newOli.Age_Range_Low_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Age_Range_Low_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Age_Range_Low_Linkedin__c + ' to ' + newOli.Age_Range_Low_Linkedin__c + '\n';
    }
    if (oldOli.Age_Range_High_Linkedin__c != newOli.Age_Range_High_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Age_Range_High_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Age_Range_High_Linkedin__c + ' to ' + newOli.Age_Range_High_Linkedin__c + '\n';
    }
    if (oldOli.Gender_Linkedin__c != newOli.Gender_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Gender_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Gender_Linkedin__c + ' to ' + newOli.Gender_Linkedin__c + '\n';
    }
    if (oldOli.Regions_States_Province_Linkedin__c != newOli.Regions_States_Province_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Regions_States_Province_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Regions_States_Province_Linkedin__c + ' to ' + newOli.Regions_States_Province_Linkedin__c + '\n';
    }
    if (oldOli.Cities_Linkedin__c != newOli.Cities_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Cities_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Cities_Linkedin__c + ' to ' + newOli.Cities_Linkedin__c + '\n';
    }
    if (oldOli.Country_Linkedin__c != newOli.Country_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Country_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Country_Linkedin__c + ' to ' + newOli.Country_Linkedin__c + '\n';
    }
    if (oldOli.DMA_Linkedin__c != newOli.DMA_Linkedin__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('DMA_Linkedin__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.DMA_Linkedin__c + ' to ' + newOli.DMA_Linkedin__c + '\n';
    }
    if (oldOli.General_Targeting_Description__c != newOli.General_Targeting_Description__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('General_Targeting_Description__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.General_Targeting_Description__c + ' to ' + newOli.General_Targeting_Description__c + '\n';
    }
    if (oldOli.Company_Categories__c != newOli.Company_Categories__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Company_Categories__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Company_Categories__c + ' to ' + newOli.Company_Categories__c + '\n';
    }
    if (oldOli.Company_Size__c != newOli.Company_Size__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Company_Size__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Company_Size__c + ' to ' + newOli.Company_Size__c + '\n';
    }
    if (oldOli.Job_Title__c != newOli.Job_Title__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Job_Title__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Job_Title__c + ' to ' + newOli.Job_Title__c + '\n';
    }
    if (oldOli.Job_Function__c != newOli.Job_Function__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Job_Function__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Job_Function__c + ' to ' + newOli.Job_Function__c + '\n';
    }
    if (oldOli.Job_Categories__c != newOli.Job_Categories__c){
      changesToNote +=  'LinkedIn ' + 
                fieldMap.get('Job_Categories__c').getDescribe().getLabel() +
                  ' changed value from ' + oldOli.Job_Categories__c + ' to ' + newOli.Job_Categories__c + '\n';
    }
    
    return changesToNote;
  }"
    
      


  


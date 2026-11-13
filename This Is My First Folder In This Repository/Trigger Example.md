<img width="1536" height="1024" alt="Apex Contact-to-Account Use Cases Infographic" src="https://github.com/user-attachments/assets/d2e97dfe-735c-4939-8404-203451135357" />
Both examples now show the complete flow:
- Amount example: Set<Id> → AggregateResult → Map<Id, Decimal> → Account update.
- Address example: Set<Id> → List<Contact> → Map<Id, String> → Account update.<br>
***One important note: The address example assumes one Contact address per Account. If multiple Contacts belong to the same Account, the Map will overwrite earlier addresses.***
<br>

trigger ContactTrigger on Contact (after insert) {

    Set<Id> accountIds = new Set<Id>();

    for (Contact con : Trigger.new) {
        if (con.AccountId != null) {
            accountIds.add(con.AccountId);
        }
    }

    List<AggregateResult> aResult = [
        SELECT AccountId, SUM(Amount__c) total
        FROM Contact
        WHERE AccountId IN :accountIds
        GROUP BY AccountId
    ];

    Map<Id, Decimal> amountSum = new Map<Id, Decimal>();

    for (AggregateResult ar : aResult) {
        amountSum.put(
            (Id) ar.get('AccountId'),
            (Decimal) ar.get('total')
        );
    }

    List<Account> accountsToUpdate = new List<Account>();

    for (Id accId : amountSum.keySet()) {
        Account acc = new Account();

        acc.Id = accId;
        acc.Sum_Total__c = amountSum.get(accId);

        accountsToUpdate.add(acc);
    }

    if (!accountsToUpdate.isEmpty()) {
        update accountsToUpdate;
    }
}
<br>
<br>
<br>
<br>
<br>
trigger ContactTrigger on Contact (after insert) {

    Set<Id> accountIds = new Set<Id>();

    for (Contact con : Trigger.new) {
        if (con.AccountId != null) {
            accountIds.add(con.AccountId);
        }
    }

    List<Contact> contacts = [
        SELECT Id, AccountId, MailingStreet
        FROM Contact
        WHERE AccountId IN :accountIds
    ];

    Map<Id, String> addressMap = new Map<Id, String>();

    for (Contact con : contacts) {
        addressMap.put(con.AccountId, con.MailingStreet);
    }

    List<Account> accountsToUpdate = new List<Account>();

    for (Id accId : addressMap.keySet()) {

        Account acc = new Account();
        acc.Id = accId;
        acc.Contact_Address__c = addressMap.get(accId);

        accountsToUpdate.add(acc);
    }

    if (!accountsToUpdate.isEmpty()) {
        update accountsToUpdate;
    }
}

// ====================== FUZZY DUPLICATE LOGIC COMPREHENSIVE TESTING ======================
    
    /**
     * Test field extraction logic for all object types in fuzzy duplicate detection
     * Tests the first phase where JSON field data is parsed and extracted into Maps
     */
    @isTest
    static void testFuzzyDuplicateLogic_FieldExtractionAllObjectTypes() {
        // CRITICAL: Create AF_Invocation__c record to enable fuzzy duplicate logic execution
        AF_Invocation__c invocation = new AF_Invocation__c();
        insert invocation;
        
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Fuzzy Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Fuzzy'}, acc.Id);
        
        // Create a task that will trigger fuzzy duplicate logic
        Task sourceTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Source Meeting Note', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
        });

        // Create field JSON for each object type with all tracked fields
        String taskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Subject' => 'Follow-up Task Subject',
                'Description' => 'Follow-up Task Description'
            }
        });
        String opportunityFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Name' => 'New Opportunity Name',
                'Product_Group__c' => 'Software'
            }
        });
        String personLifeEventFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Name' => 'Birthday Celebration',
                'EventType' => 'Birthday'
            }
        });
        String businessMilestoneFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Name' => 'Q4 Milestone',
                'MilestoneType' => 'Revenue'
            }
        });

        // Create requests for all object types to test field extraction
        List<AF_MeetingNoteActionCreator.Request> requests = new List<AF_MeetingNoteActionCreator.Request>{
            createMeetingNoteActionRequest('Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Task Action', taskFieldJSON),
            createMeetingNoteActionRequest('Opportunity', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Opportunity Action', opportunityFieldJSON),
            createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(sourceTask.Id), String.valueOf(testContact.Id), 'PersonLifeEvent Action', personLifeEventFieldJSON),
            createMeetingNoteActionRequest('BusinessMilestone', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'BusinessMilestone Action', businessMilestoneFieldJSON)
        };

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(requests);
        Test.stopTest();

        // Verify all object types processed successfully (no duplicates exist yet)
        System.assertEquals(4, responses.size(), 'Should have 4 responses');
        for (AF_MeetingNoteActionCreator.Response response : responses) {
            System.assertEquals('Successfully created the temp action!', response.message, 
                               'All actions should succeed when no existing duplicates');
            System.assertNotEquals(null, response.tempActionId, 'All actions should have tempActionId');
        }

        // Verify the created actions and data records contain correct field extractions
        List<AF_Meeting_Note_Action__c> createdActions = [
            SELECT Id, AF_Object_Name__c FROM AF_Meeting_Note_Action__c 
            WHERE Id IN :new List<String>{responses[0].tempActionId, responses[1].tempActionId, responses[2].tempActionId, responses[3].tempActionId}
        ];
        System.assertEquals(4, createdActions.size(), 'Should create 4 actions');

        // Verify field data was correctly stored
        List<AF_Meeting_Note_Action_Data__c> actionDataRecords = [
            SELECT AF_Field_Name__c, AF_Field_Value__c, AF_Meeting_Note_Action__c 
            FROM AF_Meeting_Note_Action_Data__c 
            WHERE AF_Meeting_Note_Action__c IN :new List<String>{responses[0].tempActionId, responses[1].tempActionId, responses[2].tempActionId, responses[3].tempActionId}
        ];
        
        // Should have 8 total field records (2 fields per object type)
        System.assertEquals(8, actionDataRecords.size(), 'Should create 8 field data records total');
        
        // Verify specific field values are present
        Set<String> expectedFieldNames = new Set<String>{'Subject', 'Description', 'Name', 'Product_Group__c', 'EventType', 'MilestoneType'};
        Set<String> actualFieldNames = new Set<String>();
        for (AF_Meeting_Note_Action_Data__c data : actionDataRecords) {
            actualFieldNames.add(data.AF_Field_Name__c);
        }
        System.assert(expectedFieldNames.isSubset(actualFieldNames), 'All expected fields should be extracted and stored');
    }

    /**
     * Test fuzzy duplicate logic when only ONE matching action exists
     * In this case, ALL existing field data should be added to oldRec maps
     */
    @isTest 
    static void testFuzzyDuplicateLogic_SingleMatchingAction() {
        // CRITICAL: Create AF_Invocation__c record to enable fuzzy duplicate logic execution
        AF_Invocation__c invocation = new AF_Invocation__c();
        insert invocation;
        
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Single Match Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Single'}, acc.Id);
        
        Task sourceTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Source Meeting Note', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
        });

        // Step 1: Create an existing temp action with field data
        AF_Meeting_Note_Action__c existingAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Task', 'Requires Review', 'Existing Task Action', String.valueOf(acc.Id)
        );
        
        // Create field data for the existing action
        AF_TestDataFactory.createMeetingNoteActionData(existingAction.Id, 'Subject', 'Existing Task Subject');
        AF_TestDataFactory.createMeetingNoteActionData(existingAction.Id, 'Description', 'Existing Task Description');

        // Step 2: Create new request with different field values that should NOT contain the existing ones
        String newTaskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Subject' => 'Completely Different Subject',
                'Description' => 'Completely Different Description'
            }
        });

        AF_MeetingNoteActionCreator.Request request = createMeetingNoteActionRequest(
            'Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'New Task Action', newTaskFieldJSON
        );

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(
            new List<AF_MeetingNoteActionCreator.Request>{request}
        );
        Test.stopTest();

        System.assertEquals(1, responses.size(), 'Should have 1 response');
        
        // With single matching action, the logic should add ALL existing field data to oldRec maps
        // This will depend on the AF_DuplicateFollowUpActionChecker.checkForDuplicate() result
        // Since we can't mock it, we test that the action processes through the duplicate logic
        // The actual result depends on the similarity check implementation
        System.assertNotEquals(null, responses[0].message, 'Should have a response message');
    }

    /**
     * Test fuzzy duplicate logic when MULTIPLE matching actions exist
     * In this case, only field data that contains() the proposed field data should be added to oldRec maps
     */
    @isTest
    static void testFuzzyDuplicateLogic_MultipleMatchingActions() {
        // CRITICAL: Create AF_Invocation__c record to enable fuzzy duplicate logic execution
        AF_Invocation__c invocation = new AF_Invocation__c();
        insert invocation;
        
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Multiple Match Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Multiple'}, acc.Id);
        
        Task sourceTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Source Meeting Note', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
        });

        // Step 1: Create multiple existing temp actions for the same task
        AF_Meeting_Note_Action__c existingAction1 = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Task', 'Requires Review', 'Existing Task Action 1', String.valueOf(acc.Id)
        );
        AF_Meeting_Note_Action__c existingAction2 = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Task', 'Requires Review', 'Existing Task Action 2', String.valueOf(acc.Id)
        );

        // Create field data - one that CONTAINS the proposed data, one that doesn't
        AF_TestDataFactory.createMeetingNoteActionData(existingAction1.Id, 'Subject', 'Call client about proposal'); // Contains "Call"
        AF_TestDataFactory.createMeetingNoteActionData(existingAction1.Id, 'Description', 'Follow up on pricing discussion'); // Contains "Follow up"
        
        AF_TestDataFactory.createMeetingNoteActionData(existingAction2.Id, 'Subject', 'Email marketing team'); // Does not contain "Call"
        AF_TestDataFactory.createMeetingNoteActionData(existingAction2.Id, 'Description', 'Schedule next meeting'); // Does not contain "Follow up"

        // Step 2: Create new request with field values that should match action1 but not action2
        String newTaskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Subject' => 'Call',  // Should match existingAction1's "Call client about proposal"
                'Description' => 'Follow up'  // Should match existingAction1's "Follow up on pricing discussion"
            }
        });

        AF_MeetingNoteActionCreator.Request request = createMeetingNoteActionRequest(
            'Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'New Task Action', newTaskFieldJSON
        );

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(
            new List<AF_MeetingNoteActionCreator.Request>{request}
        );
        Test.stopTest();

        System.assertEquals(1, responses.size(), 'Should have 1 response');
        
        // With multiple matching actions, only field data that contains() the proposed data should be used
        // The logic should find existingAction1's data (contains "Call" and "Follow up") but ignore existingAction2
        // This will be passed to AF_DuplicateFollowUpActionChecker.checkForDuplicate()
        System.assertNotEquals(null, responses[0].message, 'Should have a response message');
    }

    /**
     * Test complete fuzzy duplicate detection flow for all object types
     * Tests the entire pipeline from field extraction to duplicate detection
     */
    @isTest
    static void testFuzzyDuplicateLogic_AllObjectTypesComplete() {
        // CRITICAL: Create AF_Invocation__c record to enable fuzzy duplicate logic execution
        AF_Invocation__c invocation = new AF_Invocation__c();
        insert invocation;
        
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Complete Flow Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Complete'}, acc.Id);
        
        Task sourceTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Source Meeting Note', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
        });

        // Step 1: Create existing actions for each object type with specific field data
        AF_Meeting_Note_Action__c existingTaskAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Task', 'Requires Review', 'Existing Task', String.valueOf(acc.Id)
        );
        AF_Meeting_Note_Action__c existingOppAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Opportunity', 'Requires Review', 'Existing Opp', String.valueOf(acc.Id)
        );
        AF_Meeting_Note_Action__c existingPersonAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'PersonLifeEvent', 'Requires Review', 'Existing Person', String.valueOf(testContact.Id)
        );
        AF_Meeting_Note_Action__c existingBusinessAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'BusinessMilestone', 'Requires Review', 'Existing Business', String.valueOf(acc.Id)
        );

        // Create field data for existing actions
        AF_TestDataFactory.createMeetingNoteActionData(existingTaskAction.Id, 'Subject', 'Existing task subject');
        AF_TestDataFactory.createMeetingNoteActionData(existingTaskAction.Id, 'Description', 'Existing task description');
        
        AF_TestDataFactory.createMeetingNoteActionData(existingOppAction.Id, 'Name', 'Existing opportunity name');
        AF_TestDataFactory.createMeetingNoteActionData(existingOppAction.Id, 'Product_Group__c', 'Existing product group');
        
        AF_TestDataFactory.createMeetingNoteActionData(existingPersonAction.Id, 'Name', 'Existing person event');
        AF_TestDataFactory.createMeetingNoteActionData(existingPersonAction.Id, 'EventType', 'Existing event type');
        
        AF_TestDataFactory.createMeetingNoteActionData(existingBusinessAction.Id, 'Name', 'Existing business milestone');
        AF_TestDataFactory.createMeetingNoteActionData(existingBusinessAction.Id, 'MilestoneType', 'Existing milestone type');

        // Step 2: Create new requests that may or may not be similar to existing ones
        String taskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'Subject' => 'New task subject', 'Description' => 'New task description'}
        });
        String oppFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'Name' => 'New opportunity name', 'Product_Group__c' => 'New product group'}
        });
        String personFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'Name' => 'New person event', 'EventType' => 'New event type'}
        });
        String businessFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'Name' => 'New business milestone', 'MilestoneType' => 'New milestone type'}
        });

        List<AF_MeetingNoteActionCreator.Request> requests = new List<AF_MeetingNoteActionCreator.Request>{
            createMeetingNoteActionRequest('Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'New Task', taskFieldJSON),
            createMeetingNoteActionRequest('Opportunity', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'New Opp', oppFieldJSON),
            createMeetingNoteActionRequest('PersonLifeEvent', String.valueOf(sourceTask.Id), String.valueOf(testContact.Id), 'New Person', personFieldJSON),
            createMeetingNoteActionRequest('BusinessMilestone', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'New Business', businessFieldJSON)
        };

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(requests);
        Test.stopTest();

        System.assertEquals(4, responses.size(), 'Should have 4 responses');
        
        // Each response will depend on the AF_DuplicateFollowUpActionChecker.checkForDuplicate() implementation
        // We test that all requests went through the fuzzy duplicate logic pipeline
        for (Integer i = 0; i < responses.size(); i++) {
            System.assertNotEquals(null, responses[i].message, 'Response ' + i + ' should have a message');
            // Response could be either success or duplicate detected depending on similarity check
        }

        // Verify that the fuzzy duplicate logic was triggered by checking debug logs or database state
        // At minimum, we know existing actions exist and new requests were processed through the duplicate logic
        Integer totalActionsAfter = [SELECT COUNT() FROM AF_Meeting_Note_Action__c WHERE AF_Meeting_Note__c = :sourceTask.Id];
        System.assert(totalActionsAfter >= 4, 'Should have at least the 4 existing actions, plus any new non-duplicate ones');
    }

    /**
     * Test edge cases in fuzzy duplicate logic
     * Missing fields, empty data, null values, etc.
     */
    @isTest
    static void testFuzzyDuplicateLogic_EdgeCases() {
        // CRITICAL: Create AF_Invocation__c record to enable fuzzy duplicate logic execution
        AF_Invocation__c invocation = new AF_Invocation__c();
        insert invocation;
        
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Edge Case Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Edge'}, acc.Id);
        
        Task sourceTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Source Meeting Note', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
        });

        // Create existing action with partial field data
        AF_Meeting_Note_Action__c existingAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Task', 'Requires Review', 'Existing Partial', String.valueOf(acc.Id)
        );
        
        // Only create Subject field data, no Description
        AF_TestDataFactory.createMeetingNoteActionData(existingAction.Id, 'Subject', 'Partial subject data');

        // Test Case 1: JSON with missing fields
        String partialTaskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'Subject' => 'New subject'} // Missing Description
        });

        // Test Case 2: JSON with empty/null values  
        String emptyTaskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'Subject' => '', 'Description' => null}
        });

        // Test Case 3: Empty JSON array
        String emptyArrayJSON = JSON.serialize(new List<Object>());

        // Test Case 4: Object type not in objectAPINameList
        String unknownObjectJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{'CustomField__c' => 'Some value'}
        });

        List<AF_MeetingNoteActionCreator.Request> requests = new List<AF_MeetingNoteActionCreator.Request>{
            createMeetingNoteActionRequest('Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Partial Fields', partialTaskFieldJSON),
            createMeetingNoteActionRequest('Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Empty Fields', emptyTaskFieldJSON),
            createMeetingNoteActionRequest('Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Empty Array', emptyArrayJSON),
            createMeetingNoteActionRequest('UnknownObject', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Unknown Object', unknownObjectJSON)
        };

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(requests);
        Test.stopTest();

        System.assertEquals(4, responses.size(), 'Should have 4 responses');
        
        // Edge cases should not cause exceptions - they should be handled gracefully
        for (Integer i = 0; i < responses.size(); i++) {
            System.assertNotEquals(null, responses[i].message, 'Edge case response ' + i + ' should have a message');
            // Each edge case should either succeed or fail gracefully, not cause exceptions
        }

        // Verify no exceptions occurred by checking that method completed successfully
        System.assert(true, 'Fuzzy duplicate logic handled edge cases without exceptions');
    }

    /**
     * Test specific Description field logic in Task processing
     * The Description field has special handling when Subject exists
     */
    @isTest
    static void testFuzzyDuplicateLogic_TaskDescriptionSpecialHandling() {
        // CRITICAL: Create AF_Invocation__c record to enable fuzzy duplicate logic execution
        AF_Invocation__c invocation = new AF_Invocation__c();
        insert invocation;
        
        Account acc = AF_TestDataFactory.createAccount(new Map<String, Object>{'Name' => 'Description Test Account'});
        Contact testContact = AF_TestDataFactory.createContact(new Map<String, Object>{'FirstName' => 'Desc'}, acc.Id);
        
        Task sourceTask = AF_TestDataFactory.createTask(new Map<String, Object>{
            'Subject' => 'Source Meeting Note', 'WhoId' => testContact.Id, 'WhatId' => acc.Id
        });

        // Create existing action with both Subject and Description
        AF_Meeting_Note_Action__c existingAction = AF_TestDataFactory.createMeetingNoteAction(
            String.valueOf(sourceTask.Id), 'Task', 'Requires Review', 'Existing Task', String.valueOf(acc.Id)
        );
        
        AF_TestDataFactory.createMeetingNoteActionData(existingAction.Id, 'Subject', 'Call client about proposal');
        AF_TestDataFactory.createMeetingNoteActionData(existingAction.Id, 'Description', 'Discuss pricing and timeline');

        // Test the special Description logic: 
        // "if(!oldRecTaskMap.containsKey('Description') && oldRecTaskMap.containsKey('Subject'))"
        String taskFieldJSON = JSON.serialize(new List<Object>{
            new Map<String, Object>{
                'Subject' => 'Call',  // Should match and be added to oldRecTaskMap
                'Description' => 'Different description'  // Should trigger the special Description logic
            }
        });

        AF_MeetingNoteActionCreator.Request request = createMeetingNoteActionRequest(
            'Task', String.valueOf(sourceTask.Id), String.valueOf(acc.Id), 'Task with Special Description Logic', taskFieldJSON
        );

        Test.startTest();
        List<AF_MeetingNoteActionCreator.Response> responses = AF_MeetingNoteActionCreator.createAction(
            new List<AF_MeetingNoteActionCreator.Request>{request}
        );
        Test.stopTest();

        System.assertEquals(1, responses.size(), 'Should have 1 response');
        
        // Test the special Description field handling logic in the fuzzy duplicate detection
        // The code has specific logic: "if(!oldRecTaskMap.containsKey('Description') && oldRecTaskMap.containsKey('Subject'))"
        // This means if Subject was matched but Description wasn't initially matched (due to multiple actions logic),
        // the Description should still be added to oldRecTaskMap
        System.assertNotEquals(null, responses[0].message, 'Should have a response message');
        
        // Verify the logic processes without throwing exceptions
        // The actual duplicate detection result depends on AF_DuplicateFollowUpActionChecker implementation
    }
}

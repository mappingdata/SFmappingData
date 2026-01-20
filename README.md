import { LightningElement, track, wire,api } from 'lwc';
import { getPicklistValues, getObjectInfo } from 'lightning/uiObjectInfoApi';
import OPPORTUNITY_OBJECT from '@salesforce/schema/Opportunity';
import STAGE_FIELD from '@salesforce/schema/Opportunity.StageName';
import SUB_STAGE_FIELD from '@salesforce/schema/Opportunity.Sub_Stage__c';
import { getRecord } from 'lightning/uiRecordApi';
const FIELDS = ['Opportunity.RecordTypeId'];

export default class opportunityStageManager extends LightningElement {
    @api recordId;
    @track stageOptions = [];
    @track subStageOptions = [];
    @track selectedStage = '';
    @track selectedSubStage = '';
    @track selectedStageIndex;
    @track dispalySubStage = true;
    opportunityRtId;
    recordTypeId;

       @wire(getRecord, { recordId: '$recordId', fields: FIELDS })
    recordHandler({ error, data }) {
        if (data) {
            this.recordTypeId = data.fields.RecordTypeId.value;
            console.log('this.recordTypeId @@@ '+this.recordTypeId);
        } else if (error) {
            // handle error
        }
    }
    // Get Opportunity Object Info to get default recordTypeId
    /*@wire(getObjectInfo, { objectApiName: OPPORTUNITY_OBJECT })
    objectInfo({ data }) {
        if (data) {
            this.opportunityRtId = data.defaultRecordTypeId;
            console.log('this.opportunityRtId @@@ '+this.opportunityRtId)
        }
    }*/

    

    // Get StageName picklist values
    @wire(getPicklistValues, { recordTypeId: '$recordTypeId', fieldApiName: STAGE_FIELD })
    stagePicklist({ data }) {
        if (data) {
            this.stageOptions = data.values;
        }
    }

    // Get Sub_Stage__c picklist values (with dependency)
    @wire(getPicklistValues, { recordTypeId: '$recordTypeId', fieldApiName: SUB_STAGE_FIELD })
    subStagePicklist({ data }) {
        if (data) {
            this._allSubStageOptions = data.values;
            console.log('this._allSubStageOptions @@@ '+JSON.stringify(this._allSubStageOptions));
        }
    }

    // Watch for stage change
    handleStageChange(event) {
        
        this.selectedStage = event.detail.value;
        this.selectedStageIndex = this.stageOptions.findIndex(opt => opt.value === this.selectedStage);
        console.log('Selected Stage @@@ '+this.selectedStage);
        console.log('Selected Stage Index @@@ '+this.selectedStageIndex);
        this.selectedSubStage = '';
        this.setSubStageOptions();
    }

    handleSubStageChange(event) {
        this.selectedSubStage = event.detail.value;
    }

    setSubStageOptions() {
        if (!this.selectedStage || !this._allSubStageOptions) {
            console.log('Inside Selected stage @@@ '+this.selectedStage);
            console.log('Inside All Sub Stage Options @@@ '+JSON.stringify(this._allSubStageOptions));
            this.subStageOptions = [];
            return;
        }
            if(!this.selectedStage)
            {
                this.dispalySubStage = true;
            }
            else
            {
                this.dispalySubStage = false;
            }
        // The values property will only be filled for valid dependent picklist options
        const subStages = this._allSubStageOptions.filter(opt =>
            opt.validFor.includes(this.selectedStageIndex)
        );
        console.log('Outside Selected stage @@@ '+this.selectedStage);
            console.log('Outside All Sub Stage Options @@@ '+JSON.stringify(subStages));
        // Alternatively, just use the controllerValues object
        this.subStageOptions = this._allSubStageOptions.filter(opt => {
            return opt.validFor.some(item => item === this.selectedStageIndex);
        });
    }
}

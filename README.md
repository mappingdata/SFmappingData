.custom-card {
    background: #fff;
    border-radius: 4px;
    box-shadow: 0 1px 2px rgba(0,0,0,0.10);
    padding: 0 0 8px 0;
    max-width: 100%;
}

.header-row {
    display: flex;
    align-items: center;
    padding: 16px 0 8px 12px;
    font-weight: bold;
}

.crown-icon-sizer {
    display: inline-block;
    margin-right: 7px;
}

.title {
    font-size: 1.12rem;
    color: #222;
}

.badge-completed {
    background: #43B02A;
    color: #fff;
    padding: 2px 14px;
    border-radius: 12px;
    font-size: 14px;
    font-weight: 500;
    display: inline-block;
}

.badge-canceled {
    background: #DA291C;
    color: #fff;
    padding: 2px 14px;
    border-radius: 12px;
    font-size: 14px;
    font-weight: 500;
    display: inline-block;
}

.sf-link {
    color: #007CB0;
    text-decoration: underline;
    cursor: pointer;
}
.sf-viewall {
    padding-top: 10px;
    text-align: center;
}
.sf-viewall .sf-link {
    font-size: 16px;
    font-weight: 400;
}

//Css End

<template>
    <div class="custom-card">
        <div class="header-row">
            <!-- Inline SVG Red Crown Icon -->
            <span class="crown-icon-sizer">
                <svg viewBox="0 0 24 24" width="22" height="22" style="vertical-align:middle">
                    <rect width="24" height="24" rx="4" fill="#DA291C"/>
                    <path d="M4.8,12.6l2.95-6.06c.25-.51,1-.52,1.26,0L12,11.32l2.99-4.78c.27-.44,1-.45,1.26,0l2.95,6.06c.37.75-.54,1.5-1.14.97l-1.98-1.74-1.8,5.73c-.13.41-.78.4-.91,0l-1.8-5.73-1.98,1.74c-.61.53-1.51-.22-1.14-.97ZM6,17v-1h12v1c0,.55-.45,1-1,1H7c-.55,0-1-.45-1-1Z" fill="#fff"/>
                </svg>
            </span>
            <span class="title">Opportunities</span>
        </div>

        <lightning-datatable
            key-field="id"
            data={opportunities}
            columns={columns}
            hide-checkbox-column="false"
            show-row-number-column="false">
        </lightning-datatable>

        <div class="sf-viewall">
            <a href={viewAllUrl} target="_blank" class="sf-link">View All</a>
        </div>
    </div>
</template>

// HTML End
import { LightningElement } from 'lwc';


export default class OpportunitiesTable extends LightningElement {
    viewAllUrl = '/lightning/o/Opportunity/list?filterName=Recent';

    // Custom datatable columns
    columns = [
        {
            label: 'Status',
            fieldName: 'status',
            type: 'customStatus',
            typeAttributes: { status: { fieldName: 'status' } },
            cellAttributes: {
                class: { fieldName: 'statusClass' }
            }
        },
        {
            label: 'Opportunity Name',
            fieldName: 'link',
            type: 'url',
            typeAttributes: {
                label: { fieldName: 'name' },
                target: '_blank'
            },
            cellAttributes: { class: 'sf-link' }
        },
        { label: 'Date', fieldName: 'date', type: 'date' },
        { label: 'Amount', fieldName: 'amount', type: 'text' },
        { label: 'Created By', fieldName: 'createdBy', type: 'text' }
    ];

    opportunities = [
        {
            id: 1,
            status: 'Completed',
            statusClass: 'badge-completed',
            name: 'Michael Davis - Estate Planning Consultation',
            link: '#',
            date: '2025-11-20',
            amount: '$70,000',
            createdBy: 'Carlton'
        },
        {
            id: 2,
            status: 'Canceled',
            statusClass: 'badge-canceled',
            name: 'Jane Client - Life Insurance Review/Opportunity',
            link: '#',
            date: '2024-09-20',
            amount: '$50,000',
            createdBy: 'Carlton'
        },
        {
            id: 3,
            status: 'Completed',
            statusClass: 'badge-completed',
            name: 'Sarah Miller - Investment Portfolio Review',
            link: '#',
            date: '2025-06-04',
            amount: '$10,000',
            createdBy: 'Carlton'
        },
        {
            id: 4,
            status: 'Canceled',
            statusClass: 'badge-canceled',
            name: 'Andrew Wilson - Mortgage Protection Quote',
            link: '#',
            date: '2025-07-08',
            amount: '$10,000',
            createdBy: 'Carlton'
        }
    ];
}

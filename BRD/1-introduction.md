# 1. Introduction

The Lead Time Matrix consists of two key components:

1. A **configuration component**, which allows the business to define and manage lead times based on variables such as product, branding method, quantity, and production characteristics.

2. A **calculation engine**, which applies defined rules to determine the final lead time for both jobcards and sales orders.

This functionality must be implemented within Moyo as the central system responsible for lead time management. Other systems will consume these lead times and must therefore integrate the Lead Time Matrix to ensure consistency across the business.

## Required Integrations

The following integrations are required:

### 1. Amtrack

- Lead time calculation for Orders and Jobcards
- Support for the New Production Report

### 2. Website

- Branding calculator for customer-facing lead time estimates

### 3. Moyo

- Branding calculator within the Stock Checking module

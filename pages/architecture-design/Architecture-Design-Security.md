
# Security Architecture Design

## Overview

Security architecture design is a critical aspect

## Domain Controller Policy

For this iteration, we will focus on Host-Based Access Control (HBAC) as a key component of security architecture design. HBAC is a method of controlling access to resources based on the host from which the request originates.

It allows administrators to define policies that restrict access to services and resources based on the host's identity, enhancing security by ensuring that only authorized hosts can access specific services.

We are going to use a fine-grained approach to HBAC, adding rules for each service we have in our infrastructure, so that we can add this rules only to the hosts and users that need to access the service in granular way.

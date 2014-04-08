# CDF Integration

Project Clearwater deployments include a cluster of [Ralf](https://github.com/Metaswitch/ralf) nodes.  The nodes provide an HTTP interface to Sprout and Bono on which they can report billable events.  Ralf then acts as a CTF (Charging Triggering Function) and may pass these events on to a configured CDF (Charging Data Function) over the [Rf interface](http://www.3gpp.org/DynaReport/32299.htm).

By default, having spun up a Clearwater deployment, either manually or through the automated Chef install process, your deployment is not configured with a CDF and thus will not generate billing records.  This document describes how the CDF is chosen and used and how to integrate your Project Clearwater deployment with a CDF and thus enable Rf billing.

## How it works

When Sprout or Bono have handled an item of work for a given subscriber, they generate a record of that work and transmit it to the Ralf cluster for billing to the CDF.  This record will be used by Ralf to generate an Rf billing ACR (Accounting-Request) message.

To determine which CDF to send the ACR to, the node acting as the P-CSCF/IBCF is responsible for adding a `P-Charging-Function-Addresses` header to all SIP messages it proxies into the core.  This header contains a prioritised list of CDFs to send ACRs to.

Bono supports adding this header when acting as either a P-CSCF or IBCF and configuring the contents of this header is the only requirement for enabling Rf billing in your deployment.

## How to enable it

This section discusses how to enable Rf billing to a given CDF.

### Before you begin

Before connecting your deployment to a CDF, you must

*   [install Clearwater](Installation Instructions)
*   install an external CDF - details for this will vary depending on which CDF you are using.
*   ensure your CDF's firewall allows incoming connections from the nodes in the Ralf cluster on the DIAMETER port (default 3868).
 
### Connecting to a CDF

If you have a CDF set up to receive Rf billing messages from your deployment, you will need to modify the `/etc/clearwater/config` file on your Bono node to contain the following line:

    cdf_address=<CDF DIAMETER Identity>

Once you have done this, run the following command to cause Bono to pick up the changes.

    sudo monit restart bono

## Restrictions

The very first release of Ralf, from the Counter-Strike release of Project Clearwater, does not generate Rf billing messages since the related changes to Sprout and Bono (to report billable events) were not enabled.  This version was released to allow systems integrators to get a head start on spinning up and configuring Ralf nodes rather than having to wait for the next release.
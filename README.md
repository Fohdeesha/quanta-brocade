
# quanta-brocade
Documentation for flashing Brocade TurboIron firmware to Quanta LB6M 
(or flashing LB6M Fastpath firmware to a Brocade TurboIron using the revert guide)

The raw markdown source in this repo is intended to be built using [MkDocs](http://www.mkdocs.org/) - not displayed on GitHub.

Guide: [brokeaid.com](http://brokeaid.com/)  

STH Discussion Thread - [ServeTheHome](https://forums.servethehome.com/index.php?threads/turbocharge-your-quanta-lb6m-flash-to-brocade-turboiron.17971/)  

Q. Why does this work without modifying the firmware?  

A. When Broadcom introduced the BCM56820 switch ASIC, they released a reference design - a suggested switch design implementing said ASIC. Several companies including Brocade and Quanta took this reference design, built it, added their own branding, and sold it as a product. Therefore the switches are nearly identical, aside from small differences like LED configuration. They both have:

 - MPC8541 PowerPC Management CPU
 - BCM56820 10gbE Switching ASIC
 - BCM5482 Management Port PHY
 - BCM5464 Copper Ports PHY
 
 Because of this, firmware written for one platform will have no issue running on the other - it's the same hardware.
 
Open to PR's! If you know of any other switch pairs that are the same underlying hardware and would like to cross-flash, let us know! We're mainly seeking firmware candidates for the LB4M - so far we can't find any other switches with the same hardware configuration. We need to find a switch that uses a MPC8541 management CPU + BCM56514 ASIC
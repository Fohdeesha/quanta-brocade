# Backstory

## How does this work without firmware modification?
When Broadcom introduced the BCM56820 switch ASIC, they released a reference design - a suggested switch design implementing said ASIC. Several companies including Brocade, Dell, and Quanta took this reference design, built it, added their own branding, and sold it as a product. Even Google used this reference design several years later for their Pluto switch, but they were never sold. Therefore the switches are nearly identical, aside from small differences like LED configuration. They have:

 - MPC8541 PowerPC Management CPU
 - BCM56820 10gbE Switching ASIC
 - BCM5482 Management Port PHY
 - BCM5464 Copper Ports PHY
 
 Because of this, firmware written for one platform will have no issue running on the other - it's the same hardware.  
 
If you know of any other switch pairs that are the same underlying hardware and would like to cross-flash, get in touch with us on Github or the STH forum - we'll try anything.  

We're mainly seeking firmware candidates for the LB4M - so far we can't find any other switches with the same hardware configuration. We need to find a switch that uses a MPC8541 management CPU + BCM56514 ASIC.

## Why is Fastpath so quirky?
Fastpath is a [Broadcom software product](https://www.broadcom.com/products/ethernet-connectivity/software/fastpath), and the Fastpath image that comes with our switches is closer to a demo version than a full build. That's why simple things like 1gbE SFP's don't work, even though the ASIC supports them.  

When Quanta built a BCM56820 reference box, they included a barebones Fastpath build. The intent was that large customers (Microsoft & Amazon) would replace it with their own OS, or their own Fastpath builds with the exact featureset needed for their environment -  which they did. This means there's custom LB6M firmware packages floating around internally, but the chances we'll ever see them is slim to none.  

Some vendors that built and sold BCM56820 reference boxes to the public ditched Fastpath altogether and used their own OS - Brocade for example with their TurboIron 24x. Dell took a slightly different route and purchased a much more complete Fastpath base for their 8024 switch -  it uses the same ASIC as our LB6M, has a slightly faster management CPU, and runs a much more complete distribution of Fastpath. Sadly due to the CPU difference, the Dell firmware will not even boot on an LB6M (we tried).
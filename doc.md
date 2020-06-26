# Implementing multi-threading in cups-browsed
#
#### Brief Overview of cups-browsed
cups-browsed is a daemon, which discovers remote printers and makes them locally available by creating local queues for them. It provides many features to the users, such as, clustering of printers, access to printers from legacy servers, filtering of important printers out of many printers, etc. It is part of [cups-filters](https://github.com/OpenPrinting/cups-filters) and is maintained by [Open Printing](https://openprinting.github.io/).

#### Scaling Problem
As it discovers printers on the network one-by-one, and then creates queues for them. For small number of printers, it is not that time consuming. But as number of printers increases, time taken to create queues for all the printers increases a lot.
You can find more details about the time taken to create queues for different numbers of printers [here](https://github.com/mohitmo/Testing).




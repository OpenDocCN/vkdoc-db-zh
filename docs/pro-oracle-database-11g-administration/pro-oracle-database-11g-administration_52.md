# Configure environment, redundant to run each time, but ensures set correctly.
configure controlfile autobackup on;
configure controlfile autobackup format for device type disk to '/oradump01/DWREP/rman/rman_ctl_%F.bk';
configure retention policy to redundancy 1;
configure device type disk parallelism 5;
configure channel 1 device type disk format '/ora08/DWREP/rman/rman1_%U.bk';
configure channel 2 device type disk format '/ora08/DWREP/rman/rman2_%U.bk';
configure channel 3 device type disk format '/ora08/DWREP/rman/rman3_%U.bk';
configure channel 4 device type disk format '/ora09/DWREP/rman/rman4_%U.bk';
configure channel 5 device type disk format '/ora09/DWREP/rman/rman5_%U.bk';


1 Finding Failed logins (Detect brute-force)

	index=main sourcetype=access_combined status 401 
	| stats count by clientip 
	| sort -count 

2 Spotting Spikes over time

	index=main sourcetype=access_combined
	| timechart count by status

3 Top talker/ seeing the most active IPs

	index=main sourcetype=access_combined
	| top clientip

4 filtering out the noise, keeping only what matters

	index=main sourcetype=access_combined status>=500

5 searching for a specific attacker pattern 
   
	index = main sourcetype=access_combined "union select"




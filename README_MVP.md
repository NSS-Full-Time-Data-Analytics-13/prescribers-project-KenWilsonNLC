## Prescribers Database

For this exericse, you'll be working with a database derived from the [Medicare Part D Prescriber Public Use File](https://www.hhs.gov/guidance/document/medicare-provider-utilization-and-payment-data-part-d-prescriber-0). More information about the data is contained in the Methodology PDF file. See also the included entity-relationship diagram.

1. 
    a. Which prescriber had the highest total number of claims (totaled over all drugs)? Report the npi and the total number of claims.

    SELECT nppes_provider_last_org_name AS prescriber_name, total_claim_count AS claims
    FROM prescription AS p1
	INNER JOIN prescriber AS p2
	USING (npi)
	ORDER BY claims DESC;
    -- COFFEY had the highest total number of claims--
    
    b. Repeat the above, but this time report the nppes_provider_first_name, nppes_provider_last_org_name,  specialty_description, and the total number of claims.

    SELECT nppes_provider_last_org_name AS prescriber_surname, nppes_provider_first_name AS prescriber_first_name,  total_claim_count AS claims, p2.specialty_description
    FROM prescription AS p1
	INNER JOIN prescriber AS p2
	USING (npi)
	ORDER BY claims DESC;

2. 
    a. Which specialty had the most total number of claims (totaled over all drugs)?

    SELECT DISTINCT(specialty_description), SUM(total_claim_count) AS claims
    FROM prescription AS p1
	INNER JOIN prescriber AS p2
	USING (npi)
	GROUP BY p2.specialty_description
    ORDER BY claims DESC;

    -- Family Practice had the most total number of claims

    b. Which specialty had the most total number of claims for opioids?

    SELECT specialty_description, SUM(total_claim_count) AS claims, MAX(d.opioid_drug_flag) AS opioid
    FROM prescription AS p1
    INNER JOIN prescriber AS p2
	ON p1.npi = p2.npi
    INNER JOIN drug AS d
	ON p1.drug_name = d.drug_name
    GROUP BY specialty_description
    ORDER BY claims DESC;

     -- Family Practice had the most total number of claims

    c. **Challenge Question:** Are there any specialties that appear in the prescriber table that have no associated prescriptions in the prescription table?

    d. **Difficult Bonus:** *Do not attempt until you have solved all other problems!* For each specialty, report the percentage of total claims by that specialty which are for opioids. Which specialties have a high percentage of opioids?

3. 
    a. Which drug (generic_name) had the highest total drug cost?

    SELECT drug_name, total_drug_cost_ge65 AS drug_cost
    FROM prescription
	WHERE total_drug_cost_ge65 IS NOT NULL
    ORDER BY drug_cost DESC

    -- ESBRIET had the highest total drug cost


    b. Which drug (generic_name) has the hightest total cost per day? **Bonus: Round your cost per day column to 2 decimal places. Google ROUND to see how this works.**

4. 
    a. For each drug in the drug table, return the drug name and then a column named 'drug_type' which says 'opioid' for drugs which have opioid_drug_flag = 'Y', says 'antibiotic' for those drugs which have antibiotic_drug_flag = 'Y', and says 'neither' for all other drugs. **Hint:** You may want to use a CASE expression for this. See https://www.postgresqltutorial.com/postgresql-tutorial/postgresql-case/ 

    SELECT drug_name,
    CASE
	WHEN opioid_drug_flag = 'Y' THEN 'opioid'
	WHEN antibiotic_drug_flag = 'Y' THEN 'antibiotic'
	ELSE 'neither'
	END AS drug_type
    FROM drug;

    b. Building off of the query you wrote for part a, determine whether more was spent (total_drug_cost) on opioids or on antibiotics. Hint: Format the total costs as MONEY for easier comparision.

    SELECT drug_name, total_drug_cost::money,
    CASE
	WHEN opioid_drug_flag = 'Y' THEN 'opioid'
	WHEN antibiotic_drug_flag = 'Y' THEN 'antibiotic'
	ELSE 'neither'
    END AS drug_type,
    SUM(CASE WHEN opioid_drug_flag = 'Y' THEN total_drug_cost ELSE 0 END) AS opioid_total_cost,
    SUM(CASE WHEN antibiotic_drug_flag = 'Y' THEN total_drug_cost ELSE 0 END) AS antibiotic_total_cost
    FROM drug
	INNER JOIN prescription
    USING (drug_name)
    GROUP BY drug_name, total_drug_cost, opioid_drug_flag, antibiotic_drug_flag;

5. 
    a. How many CBSAs are in Tennessee? **Warning:** The cbsa table contains information for all states, not just Tennessee.

    SELECT COUNT(*)
    FROM cbsa 
    WHERE cbsaname LIKE '%TN'

    b. Which cbsa has the largest combined population? Which has the smallest? Report the CBSA name and total population.

    SELECT c.cbsaname,
		p.population
    FROM cbsa AS c 
    LEFT JOIN population AS p
    ON c.fipscounty = p.fipscounty
    GROUP BY c.cbsaname, p.population
    ORDER BY p.population;

    smallest population: 

    c. What is the largest (in terms of population) county which is not included in a CBSA? Report the county name and population.

6. 
    a. Find all rows in the prescription table where total_claims is at least 3000. Report the drug_name and the total_claim_count.

    SELECT drug_name, total_claim_count
    FROM prescription
	WHERE total_claim_count >= 3000;

    b. For each instance that you found in part a, add a column that indicates whether the drug is an opioid.

    SELECT p.drug_name, p.total_claim_count, d.opioid_drug_flag,
	CASE 
		WHEN opioid_drug_flag = 'Y' THEN 'opioid'
		ELSE 'NA'
	END AS drug_type
    FROM prescription AS p
    LEFT JOIN drug AS d
    USING (drug_name)
    WHERE total_claim_count >= 3000;

    c. Add another column to you answer from the previous part which gives the prescriber first and last name associated with each row.

    SELECT  p2.nppes_provider_first_name, p2.nppes_provider_last_org_name, p.drug_name, p.total_claim_count, d.opioid_drug_flag, 
	CASE 
		WHEN opioid_drug_flag = 'Y' THEN 'opioid'
		ELSE 'NA'
	END AS drug_type
    FROM prescription AS p
    LEFT JOIN drug AS d
    USING (drug_name)
    LEFT JOIN prescriber AS p2
    USING (npi)
    WHERE total_claim_count >= 3000;

7. The goal of this exercise is to generate a full list of all pain management specialists in Nashville and the number of claims they had for each opioid. **Hint:** The results from all 3 parts will have 637 rows.

    a. First, create a list of all npi/drug_name combinations for pain management specialists (specialty_description = 'Pain Management) in the city of Nashville (nppes_provider_city = 'NASHVILLE'), where the drug is an opioid (opiod_drug_flag = 'Y'). **Warning:** Double-check your query before running it. You will only need to use the prescriber and drug tables since you don't need the claims numbers yet.

    SELECT p.specialty_description AS pain_management_specialists, pp.drug_name, p.npi, nppes_provider_city AS city 
    FROM prescriber AS p
    LEFT JOIN prescription AS pp
	USING (npi)
    WHERE nppes_provider_city = 'NASHVILLE';

    b. Next, report the number of claims per drug per prescriber. Be sure to include all combinations, whether or not the prescriber had any claims. You should report the npi, the drug name, and the number of claims (total_claim_count).

    SELECT p.specialty_description AS pain_management_specialists, pp.drug_name, p.npi, nppes_provider_city AS city, pp.total_claim_count
    FROM prescriber AS p
    LEFT JOIN prescription AS pp
	USING (npi)
    WHERE nppes_provider_city = 'NASHVILLE';
    
    c. Finally, if you have not done so already, fill in any missing values for total_claim_count with 0. Hint - Google the COALESCE function.
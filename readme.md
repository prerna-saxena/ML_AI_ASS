import argparse
import csv
import requests
import xml.etree.ElementTree as ET
from typing import List, Dict, Optional


class PubMedClient:
    BASE_URL = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/"

    def fetch_papers(self, query: str) -> List[str]:
        search_url = f"{self.BASE_URL}esearch.fcgi?db=pubmed&term={query}&retmax=100&retmode=xml"
        response = requests.get(search_url)
        if response.status_code != 200:
            raise Exception("Failed to fetch data from PubMed")
        root = ET.fromstring(response.content)
        return [id_elem.text for id_elem in root.findall(".//Id")]

    def fetch_paper_details(self, pubmed_id: str) -> Optional[Dict]:
        details_url = f"{self.BASE_URL}efetch.fcgi?db=pubmed&id={pubmed_id}&retmode=xml"
        response = requests.get(details_url)
        if response.status_code != 200:
            return None
        root = ET.fromstring(response.content)

        title_elem = root.find(".//ArticleTitle")
        pub_date_elem = root.find(".//PubDate")
        authors = root.findall(".//Author")

        title = title_elem.text if title_elem is not None else "Unknown"
        pub_date = pub_date_elem.find("Year").text if pub_date_elem is not None else "Unknown"

        non_academic_authors = []
        company_affiliations = []
        corresponding_email = ""

        for author in authors:
            affiliation_elem = author.find(".//Affiliation")
            email_elem = author.find(".//ElectronicAddress")

            if affiliation_elem is not None and self.is_industry_affiliation(affiliation_elem.text):
                non_academic_authors.append(
                    (author.find("ForeName").text or "") + " " + (author.find("LastName").text or "")
                )
                company_affiliations.append(affiliation_elem.text)

            if email_elem is not None:
                corresponding_email = email_elem.text

        return {
            "PubmedID": pubmed_id,
            "Title": title,
            "Publication Date": pub_date,
            "Non-academic Author(s)": ", ".join(non_academic_authors),
            "Company Affiliation(s)": ", ".join(company_affiliations),
            "Corresponding Author Email": corresponding_email,
        }

    @staticmethod
    def is_industry_affiliation(affiliation: str) -> bool:
        keywords = ["pharma", "biotech", "inc", "corp", "ltd", "gmbh", "sa"]
        return any(keyword in affiliation.lower() for keyword in keywords)


def save_to_csv(data: List[Dict], filename: str):
    if not data:
        print("No data to save.")
        return
    with open(filename, mode="w", newline="") as file:
        writer = csv.DictWriter(file, fieldnames=data[0].keys())
        writer.writeheader()
        writer.writerows(data)


def fetch_and_process_papers(query: str) -> List[Dict]:
    client = PubMedClient()
    paper_ids = client.fetch_papers(query)
    results = []
    for pubmed_id in paper_ids:
        details = client.fetch_paper_details(pubmed_id)
        if details:
            results.append(details)
    return results


def main():
    parser = argparse.ArgumentParser(description="Fetch research papers from PubMed")
    parser.add_argument("query", type=str, nargs="?", help="Search query")
    parser.add_argument("-f", "--file", type=str, help="Output CSV filename")
    parser.add_argument("-d", "--debug", action="store_true", help="Enable debug mode")

    args = parser.parse_args()

    if not args.query:
        parser.error("The following arguments are required: query")

    results = fetch_and_process_papers(args.query)

    if args.file:
        save_to_csv(results, args.file)
    else:
        for record in results:
            print(record)


if __name__ == "__main__":
    main()

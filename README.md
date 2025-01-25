Research Paper Fetcher
{
How the Code is Organized
The program is structured into the following components:
- main.py: Contains the main logic to fetch and filter research papers from PubMed based on the query.
- utils/: A folder containing helper functions for API requests, parsing responses, and CSV generation.
- tests/: Includes test cases to validate the functionality of the program.
- pyproject.toml: Used by Poetry for managing dependencies and project setup.

Instructions on How to Install Dependencies and Execute the Program
1. Clone the repository:
2. Install dependencies using Poetry:
3. Set your PubMed API key as an environment variable: 
set PUBMED_API_KEY=”your_api_key_here”
4. Run the program:
 - To fetch papers and save the results to a file: ``` poetry run get-papers-list "your query" -f output.csv ``` 
- To fetch papers and display results in the terminal: ``` poetry run get-papers-list "your query" ```
Tools and Libraries 
– PubMed API: For fetching research papers. (https://www.ncbi.nlm.nih.gov/home/develop/api/). 
- Poetry For dependency management and packaging. (https://python-poetry.org/). 
- Python Libraries: 
1.requests: For making HTTP requests to the PubMed API. 
2.beautifulsoup4: For parsing XML responses. 
3.argparse: For handling command-line arguments.
}

import os
import csv
import requests
import xml.etree.ElementTree as ET
import argparse


def fetch_paper_ids(query, api_key):
    base_url = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi"
    #enter your apikey here
    api_key = ""
    params = {
        "db": "pubmed",
        "term": query.replace(" ", "+"),
        "retmode": "xml",
        "usehistory": "y",
        "api_key": api_key,
    }
    response = requests.get(base_url, params=params)
    response.raise_for_status()
    tree = ET.fromstring(response.text)
    return [id_elem.text for id_elem in tree.findall(".//Id")]


def fetch_paper_details(paper_ids, api_key):
    base_url = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi"
    params = {
        "db": "pubmed",
        "id": ",".join(paper_ids),
        "retmode": "xml",
        "api_key": api_key,
    }
    response = requests.get(base_url, params=params)
    try:
        response.raise_for_status()
        return response.text
    except requests.exceptions.HTTPError as e:
        print(f"Error: {e}")
        print(f"Response Text: {response.text}")
        return ""


def parse_pubmed_response(xml_data):
    root = ET.fromstring(xml_data)
    results = []
    for article in root.findall(".//PubmedArticle"):
        pubmed_id = article.find(".//PMID").text if article.find(".//PMID") is not None else "N/A"
        title = article.find(".//ArticleTitle").text if article.find(".//ArticleTitle") is not None else "N/A"
        
        # Handle missing publication date
        pub_date_elem = article.find(".//PubDate")
        pub_date = (
            " ".join([child.text for child in pub_date_elem if child is not None])
            if pub_date_elem is not None else "N/A"
        )

        affiliations = []
        authors = []
        for author in article.findall(".//Author"):
            name = "{} {}".format(
                (author.find("ForeName").text or "").strip(), 
                (author.find("LastName").text or "").strip()
            ).strip()
            affiliation = author.find(".//Affiliation").text if author.find(".//Affiliation") is not None else ""
            authors.append(name)
            affiliations.append(affiliation)

        non_academic_authors = []
        company_affiliations = []
        for aff in affiliations:
            if any(keyword in aff.lower() for keyword in ["pharma", "biotech", "inc", "ltd", "corporation"]):
                company_affiliations.append(aff)
                author_index = affiliations.index(aff)
                non_academic_authors.append(authors[author_index])

        corresponding_email = None
        for aff in affiliations:
            if "@" in aff:
                corresponding_email = aff.split(" ")[-1]
                break

        results.append({
            "PubmedID": pubmed_id,
            "Title": title,
            "Publication Date": pub_date,
            "Non-academic Author(s)": "; ".join(non_academic_authors),
            "Company Affiliation(s)": "; ".join(company_affiliations),
            "Corresponding Author Email": corresponding_email or "N/A"
        })
    return results



def save_to_csv(data, filename):
    with open(filename, mode="w", newline="", encoding="utf-8") as file:
        writer = csv.DictWriter(
            file,
            fieldnames=[
                "PubmedID",
                "Title",
                "Publication Date",
                "Non-academic Author(s)",
                "Company Affiliation(s)",
                "Corresponding Author Email",
            ],
        )
        writer.writeheader()
        writer.writerows(data)


def main():
    parser = argparse.ArgumentParser(
        description="Fetch PubMed research papers based on a query."
    )
    parser.add_argument("query", type=str, help="Query string for PubMed search.")
    parser.add_argument("-f", "--file", type=str, help="Filename to save the results.")
    parser.add_argument("-d", "--debug", action="store_true", help="Enable debug mode.")
    args = parser.parse_args()

    # Ensure API key is available
    api_key = os.getenv("PUBMED_API_KEY")
    if not api_key:
        print(
            "Error: No API key found. Please set the PUBMED_API_KEY environment variable."
        )
        return

    if args.debug:
        print(f"Debug: Query - {args.query}")

    try:
        paper_ids = fetch_paper_ids(args.query, api_key)
        if args.debug:
            print(f"Debug: Retrieved {len(paper_ids)} paper IDs.")

        xml_data = fetch_paper_details(paper_ids, api_key)
        if not xml_data:
            print("Error: No data fetched from PubMed.")
            return

        results = parse_pubmed_response(xml_data)

        if args.file:
            save_to_csv(results, args.file)
            print(f"Results saved to {args.file}")
        else:
            for result in results:
                print(result)
    except Exception as e:
        print(f"Error: {e}")


if __name__ == "__main__":
    main()

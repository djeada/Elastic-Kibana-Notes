# Elastic - Kibana Notes

Practice-based learning materials designed to help you gain hands-on experience with Elasticsearch and Kibana. Through a series of labs, notes, and detailed instructions, you will learn how to perform manual REST API requests through Kibana and how to leverage Python-based automation to build scripts and workflows—preparing you for a production-like environment.

## Features

- **Detailed Labs:**  
  Step-by-step exercises integrating manual REST API requests via Kibana with Python automation, offering you a dynamic learning experience.
  
- **Hands-On Practice:**  
  Gain practical exposure by developing scripts and workflows, which helps you extend your cluster’s capabilities in realistic settings.
  
- **Comprehensive Documentation:**  
  Extensive notes supported by numerous screenshots and examples ensure that concepts are explained in a clear, engaging manner.
  
- **Real-World Scenarios:**  
  Practice operational tasks and analytics use cases common in real-world deployments—boosting both your understanding and problem-solving skills.

## About Elasticsearch

Elasticsearch is a powerful, distributed search and analytics engine built on Apache Lucene. Here’s what you need to know:

- **Core Functionality:**  
  Elasticsearch enables full-text search, log analytics, and operational intelligence with a distributed, scalable architecture.
  
- **RESTful Communication:**  
  It provides an HTTP web interface and supports schema-free JSON documents.
  
- **Open Source & Modern:**  
  Developed in Java and released under the Apache License, Elasticsearch has been widely adopted since its first release in 2010.
  
- **Common Use Cases:**  
  - Log analytics  
  - Full-text search  
  - Operational intelligence, particularly when used in conjunction with Kibana for data visualization

## Labs

| #   | Title                                                                   | Link                                                                                                                                                       |
|-----|-------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------------------------------------------------|
| 1   | Environment Setup, Cluster Basics, and Verification                     | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab01)                                                                                  |
| 2   | Index Creation, CRUD Operations, and Data Validation                    | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab02)                                                                                  |
| 3   | Basic Searching with Query DSL                                          | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab03)                                                                                  |
| 4   | Data Aggregations for Analytics                                         | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab04)                                                                                  |
| 5   | Defining Mappings and Creating Custom Analyzers                         | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab05)                                                                                  |
| 6   | Data Ingestion with Python, Logstash, and Beats                           | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab06)                                                                                  |
| 7   | Multi-Node Cluster Configuration and Monitoring                         | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab07)                                                                                  |
| 8   | Advanced Querying and Query Profiling                                   | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab08)                                                                                  |
| 9   | Modeling Relational Data with Nested Objects and Parent‑Child Relationships| [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab09)                                                                                  |
| 10  | Securing Your Elasticsearch Cluster                                     | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab10)                                                                                  |
| 11  | Dashboarding with Kibana and Python Data Visualization                    | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab11)                                                                                  |
| 12  | Performance Tuning, Snapshot Backup, and Recovery                       | [LAB](https://github.com/djeada/Elastic-Kibana-Notes/tree/main/labs/lab12)                                                                                  |

## Prerequisites

Before getting started, ensure you have the following installed and configured:

- **Elasticsearch:**  
  A running Elasticsearch cluster (either locally or on a remote server).
  
- **Kibana:**  
  Access to a Kibana instance to interact with Elasticsearch.
  
- **Python:**  
  Python 3.x installed with necessary libraries such as `requests` and `json`.
  
- **Tools & Dependencies:**  
  Git for version control, and any additional packages as specified in the project’s requirements.

## Installation

To set up the environment locally, follow these steps:

1. **Clone the Repository:**

   ```bash
   git clone https://github.com/yourusername/elastic-kibana-notes.git
   cd elastic-kibana-notes
   ```

2. **Set Up Python Environment:**

   It is recommended to create a virtual environment:
   
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows, use `venv\Scripts\activate`
   ```

3. **Install Dependencies:**

   ```bash
   pip install -r requirements.txt
   ```

4. **Configure Access:**

   Ensure your Elasticsearch and Kibana endpoints are properly configured in your environment variables or configuration files as needed by the scripts.

## Usage

Once installed, you can start exploring the repository:

- **Read the Documentation:**  
  Start with the detailed notes in the `docs` folder to understand the lab procedures and the underlying concepts.
  
- **Run Labs:**  
  Navigate to the lab directories and follow the step-by-step guides to execute REST API calls manually and through automated Python scripts.
  
- **Experiment & Extend:**  
  Use the provided examples as a base for your own experiments to enhance, automate, or extend your Elasticsearch cluster’s capabilities.

## Project Structure

Here’s an overview of the repository structure:

```
elastic-kibana-notes/
├── notes/                 # Detailed notes with screenshots
├── labs/                  # Practice labs with step-by-step guides and code samples
│   ├── lab1/              # Lab focusing on REST API interactions
│   ├── lab2/              # Lab showcasing Python automation with Elasticsearch
│   └── ...
├── scripts/               # Python scripts used for automation tasks
├── requirements.txt       # Python dependencies
├── README.md              # This file
└── LICENSE                # License information
```

## Contributing

Contributions are welcome! If you have ideas for new labs, enhancements to the documentation, or improvements to the scripts, please follow these guidelines:

1. **Fork the Repository:**  
   Create your own branch for any new features or fixes.

2. **Submit a Pull Request:**  
   Ensure your code or documentation meets the project standards and includes appropriate tests or examples.

3. **Report Issues:**  
   Use the Issues tab to report bugs or request features.

## License

This project is licensed under the [MIT License](LICENSE) - see the [LICENSE](LICENSE) file for details.

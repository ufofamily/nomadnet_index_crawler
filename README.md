NomadNet Search is a lightweight distributed indexer and local search engine for NomadNet / Reticulum nodes.
The project automatically discovers nodes through Reticulum announce packets, crawls accessible page resources (such as /page/index.mu), extracts internal links, recursively builds a local mirror of reachable content, and provides a simple Google-style web search interface over the indexed pages.
Core components:
•	Node discovery — listens for Reticulum announces and registers newly discovered destinations
•	Crawler — fetches page content from reachable NomadNet nodes
•	Recursive indexing — parses internal links and expands the crawl graph
•	Local database — stores nodes, paths, page content, and crawl state
•	Web UI — lightweight search interface for browsing indexed content
This project is designed as a local search gateway for visible parts of the NomadNet mesh rather than a global search engine for the entire network.


# Contributing to ACODE

ACODE is a production-grade asynchronous OSINT pipeline for facial co-occurrence detection. Contributions are welcome provided they align with the project’s strict engineering, ethical, and operational standards.

The project maintains a deliberately high bar for changes because outputs are intended for use in public accountability work where error rates and provenance matter.

## Development Philosophy

- All changes must preserve the **filter-then-infer** architecture and the mandatory human-in-the-loop requirement.
- No contribution may introduce automated publication, alerting, or dissemination paths.
- Performance improvements are valued only when they do not compromise auditability or calibration stability.
- Documentation and tests are mandatory for any non-trivial change.

## Code of Conduct

All contributors and maintainers are expected to follow the [Contributor Covenant](https://www.contributor-covenant.org/version/2/1/code_of_conduct/) in addition to the specific standards below.

## Development Environment

The recommended development environment is the same containerized stack used in production.

bash
git clone https://github.com/YourOrg/ACODE.git
cd ACODE
docker-compose up -d --build

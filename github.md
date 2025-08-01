TOM Proposal
Github Organization Creation
Product Team:
Identifies the need for a new GitHub Organization for a new product or major application.
Creates a Jira request (or similar ticketing system) to the Central GitHub Management Team.
Request Details: Include proposed organization name, primary owner (from product team), and a brief justification.
Central Team Activity (Review & Creation):
Central GitHub Management Team: Reviews the Jira request for adherence to organizational naming conventions and overall strategy.
Central GitHub Management Team: Creates the new GitHub Organization, assigns the requested primary owner, and configures initial global settings (e.g., billing, default visibility, basic security policies).
Product Team Activity (Initial Setup):
Product Team: Takes ownership of the newly created organization.
Product Team: Creates initial teams within their organization (e.g., "Developers," "QA," "Admins") and adds relevant members.
Repository Naming Convention
Central Team Activity (Define & Communicate):
Central GitHub Management Team: Defines and publishes a clear, mandatory repository naming convention (e.g., product-component-type, appname-service-feature).
Example: productA-frontend-web, productA-backend-auth-service, productB-mobile-ios-app.
Central GitHub Management Team: Communicates these guidelines widely to all product teams.
Product Team Activity (Adherence):
Product Team: Adheres to the defined naming conventions when creating new repositories within their organization.
Product Team: For existing repositories, they should plan to refactor names to align with the new convention if feasible and beneficial. (Requires a separate change request if renaming impacts integrations).
Branch Protection Rules
Central Team Activity (Define Templates & Enable):
Central GitHub Management Team: Defines a set of mandatory and recommended branch protection rule templates.
Mandatory: Require pull request reviews (e.g., 1 or 2 approvals), require status checks to pass before merging, require branches to be up to date before merging, restrict who can push to matching branches, enforce linear history.
Recommended: Require signed commits, include administrators in branch protection.
Central GitHub Management Team: Configures these templates to be automatically applied to new repositories or provides clear instructions for product teams to apply them.
Product Team Activity (Apply & Manage):
Product Team: Applies the mandatory branch protection rules to their main and other critical branches.
Product Team: Configures specific status checks (e.g., CI/CD build success, SonarQube scan, unit test coverage) that must pass for their repositories.
Product Team: Manages the review requirements (number of approvals) based on team size and complexity.
Roles and Responsibilities (Access Control)
Central Team Activity (Define & Configure):
Central GitHub Management Team: Defines a standardized matrix of roles and their corresponding permissions across all organizations.
Central GitHub Management Team: Configures default organization-level permissions and ensures primary product team owners are set up correctly.
Product Team Activity (Manage Teams & Members):
Product Team: Creates teams within their product organization (e.g., productA-devs, productA-qa).
Product Team: Assigns appropriate GitHub roles (Admin, Maintain, Write, Triage, Read) to these teams for each repository.
Product Team: Manages the addition and removal of individual members to these teams based on their roles and responsibilities.
Product Team: Regularly reviews team memberships and assigned permissions.
Repository Visibility (Public/Private)
Central Team Activity (Set Defaults & Policy):
Central GitHub Management Team: Sets the default repository visibility to "Private" for all new organizations and repositories.
Central GitHub Management Team: Establishes a formal policy for requesting public repositories, including a review and approval process.
Product Team Activity (Request Changes):
Product Team: Creates all new repositories as private.
Product Team: If a public repository is required (e.g., for open-source contributions, public documentation), they raise a Jira request to the Central GitHub Management Team.
Request Details: Justification for public visibility, type of content, and confirmation that no sensitive information is present.
Central Team Activity (Review & Approval):
Central GitHub Management Team: Reviews the request against the public repository policy, potentially involving security and legal teams.
Central GitHub Management Team: Approves or denies the request and changes the repository visibility if approved.
Code Review & Pull Request Strategy
Product Team Activity (Implement & Educate):
Product Team: Establishes a clear PR workflow (e.g., feature branches, PRs to develop, then merge to main).
Product Team: Educates team members on code review best practices (e.g., constructive feedback, timely reviews, clear PR descriptions).
Product Team: Utilizes GitHub's PR features: assignees, reviewers, labels, and linked issues.
Product Team: Integrates code quality tools (e.g., SonarQube, linters) into CI/CD pipelines, with results displayed in PR checks.
Security Best Practices
Central Team Activity (Enable & Monitor):
Central GitHub Management Team: Enables GitHub Advanced Security features (Dependabot, Code Scanning, Secret Scanning) at the organization level.
Central GitHub Management Team: Configures default security policies and alerts for all organizations.
Central GitHub Management Team: Establishes a central monitoring system for security alerts across all organizations.
Product Team Activity (Remediate & Integrate):
Product Team: Monitors and remediates security alerts generated by Dependabot, Code Scanning, and Secret Scanning for their repositories.
Product Team: Integrates static application security testing (SAST-Checkmarx, Trivy) and dynamic application security testing (DAST-Qualys) tools into their CI/CD pipelines.
Product Team: Implements secure coding practices and conducts regular security training.
Automation & CI/CD Integration
Central Team Activity (Provide Templates & Guidance):
Central GitHub Management Team: Provides standardized GitHub Actions workflows or templates for common CI/CD patterns (e.g., build, test, deploy to staging/production).
Central GitHub Management Team: Offers guidance and support for integrating Github Actions.
Central GitHub Management Team: Develops and implements CI/CD pipelines using GitHub Actions within product team's repositories.
Central GitHub Management Team: Configures automated tests (unit, integration, end-to-end) to run on every commit or PR.
Central GitHub Management Team: Sets up automated deployment workflows for various environments (dev, QA, staging, production).
Documentation
Product Team Activity (Create & Maintain):
Product Team: Creates a detailed README.md file in the root of each repository, covering:
Project overview and purpose
Setup instructions (local development)
Build and test commands
Deployment instructions
Contribution guidelines
Key contacts
Product Team: Utilizes GitHub Wiki or external documentation platforms (Confluence) for broader project documentation (e.g., architecture diagrams, design decisions, API specifications).
Product Team: Ensures code is well-commented, especially for complex logic or non-obvious implementations.
Product Team: Regularly reviews and updates documentation as the project evolves.
Archiving Strategy
Central Team Activity (Define Policy):
Central GitHub Management Team: Defines an archiving policy, including criteria for archiving (e.g., no activity for X months, project officially decommissioned) and the process for doing so.
Product Team Activity (Initiate & Review):
Product Team: Identifies repositories that meet the archiving criteria.
Product Team: Initiates a Jira request to the Central GitHub Management Team to archive a repository.
Request Details: Repository name, reason for archiving, and confirmation that all necessary backups or historical records have been secured.
Central Team Activity (Execute Archiving):
Central GitHub Management Team: Reviews the request.
Central GitHub Management Team: Archives the repository on GitHub, making it read-only and clearly marked as archived.

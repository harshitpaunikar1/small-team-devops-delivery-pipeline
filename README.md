# Small-Team DevOps Delivery Pipeline

This repository documents a repeatable deployment pipeline for taking an application from developer laptop to a monitored running service.

## Domain
Platform Engineering / DevOps

## Overview
Combined container packaging, CI automation, security scanning, and rollback-aware operations without overengineering the infrastructure.

## Methodology
1. Framed the deployment problem around small-team practicality, choosing Docker Compose over heavier orchestration because the system size did not justify Kubernetes.
2. Standardized runtime packaging with Docker so the app could move cleanly from local development into a consistent deployment environment.
3. Automated test, build, and release steps through Jenkins to remove fragile manual deployment work and make the process repeatable.
4. Integrated Trivy scanning before deployment so image security checks became part of the normal delivery path instead of an afterthought.
5. Added Prometheus and Grafana for health visibility, making release quality observable once the app was running in production-like conditions.
6. Designed the process with rollback and operational handover in mind so another teammate could run and recover the system later without guesswork.

## Skills
- Docker
- Docker Compose
- Jenkins
- Trivy
- Prometheus
- Grafana
- Deployment Automation
- Rollback Planning

## Source
This README was generated from the portfolio project data used by `/Users/harshitpanikar/Documents/Test_Projs/harshitpaunikar1.github.io/index.html`.

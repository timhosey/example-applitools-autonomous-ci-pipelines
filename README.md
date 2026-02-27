# Automation Pipelines for CI with Applitools Autonomous

This repository contains example of CI pipelines for Applitools Autonomous. This repo exists explicitly for testing and providing an example to those seeking to automate their Autonomous visual testing.

## Important Note!

The default timeout for waiting for a response is 60 attempts, every thirty seconds. This indicated a HALF HOUR of wait. This was put in for testing, and to sort of prove you're actually reading this (or your AI dev tool is, anyways). Change that to something more reasonable (like 5 attempts, or better yet use global timeouts for your builds) to ensure it's not just eating cycles.

## Statuses

The run fails in both Jenkins and GHA if the test aborts on the Autonomous side or visual differences or found. Use these situations to trigger automatic notifications, such as Slack, Teams, email, smoke signals or IPoAC.
# Jitter & Timing Algorithms

This package implements mathematical functions for obfuscating temporal patterns in network traffic.

## The Problem

Regular beacons (e.g., exactly every 60.0 seconds) create a distinct spike in frequency analysis (Fourier Transform), making them trivial for Blue Teams to detect using automated IDS/SIEM rules.

## The Solution

This package provides functions to calculate sleep intervals based on a randomized distribution:

$$T_{sleep} = T_{base} \pm (T_{base} \times J_{factor})$$

It serves both:

1. **The Agent**: To calculate how long to sleep.
2. **The Server**: To estimate when an agent _should_ have checked in, allowing for accurate "Lost Agent" status detection.
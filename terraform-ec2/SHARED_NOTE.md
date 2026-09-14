# Note on folder usage

This Terraform config provisions the EC2 instance shared between two assignments:
- Assignment 05: main.tf (EC2 + security group for ports 22, 80, 81)
- Assignment 06: port8082.tf (extra security group rule for Jenkins-deployed React app)

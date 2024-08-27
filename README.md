// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract ScholarshipFund {
    // Variables
    address public owner;
    uint256 public totalFunds;
    uint256 public scholarshipAmount;
    
    struct Application {
        address student;
        string name;
        string email;
        string reason;
        bool approved;
        bool paid;
    }

    Application[] public applications;
    
    mapping(address => bool) public hasApplied;

    // Events
    event DonationReceived(address donor, uint256 amount);
    event ApplicationSubmitted(address student, string name, string email, string reason);
    event ScholarshipApproved(address student, uint256 amount);
    event ScholarshipPaid(address student, uint256 amount);

    // Constructor
    constructor(uint256 _scholarshipAmount) {
        owner = msg.sender;
        scholarshipAmount = _scholarshipAmount;
    }

    // Modifier to restrict functions to the owner
    modifier onlyOwner() {
        require(msg.sender == owner, "Only the owner can perform this action");
        _;
    }

    // Function to donate funds to the scholarship
    function donate() external payable {
        require(msg.value > 0, "Donation amount must be greater than zero");

        totalFunds += msg.value;
        emit DonationReceived(msg.sender, msg.value);
    }

    // Function for students to apply for the scholarship
    function applyForScholarship(string memory _name, string memory _email, string memory _reason) external {
        require(!hasApplied[msg.sender], "You have already applied for a scholarship");

        applications.push(Application({
            student: msg.sender,
            name: _name,
            email: _email,
            reason: _reason,
            approved: false,
            paid: false
        }));

        hasApplied[msg.sender] = true;

        emit ApplicationSubmitted(msg.sender, _name, _email, _reason);
    }

    // Function for the owner to approve a scholarship application
    function approveScholarship(uint256 applicationIndex) external onlyOwner {
        require(applicationIndex < applications.length, "Invalid application index");
        require(!applications[applicationIndex].approved, "Scholarship already approved");
        require(!applications[applicationIndex].paid, "Scholarship already paid");

        applications[applicationIndex].approved = true;

        emit ScholarshipApproved(applications[applicationIndex].student, scholarshipAmount);
    }

    // Function to disburse the scholarship funds to the approved student
    function payScholarship(uint256 applicationIndex) external onlyOwner {
        require(applicationIndex < applications.length, "Invalid application index");
        require(applications[applicationIndex].approved, "Scholarship not approved");
        require(!applications[applicationIndex].paid, "Scholarship already paid");
        require(totalFunds >= scholarshipAmount, "Insufficient funds");

        applications[applicationIndex].paid = true;
        totalFunds -= scholarshipAmount;
        
        (bool success, ) = applications[applicationIndex].student.call{value: scholarshipAmount}("");
        require(success, "Transfer failed");

        emit ScholarshipPaid(applications[applicationIndex].student, scholarshipAmount);
    }

    // Function to get the number of applications
    function getApplicationCount() external view returns (uint256) {
        return applications.length;
    }

    // Function to withdraw remaining funds by the owner
    function withdrawFunds(uint256 amount) external onlyOwner {
        require(amount <= totalFunds, "Amount exceeds available funds");

        totalFunds -= amount;

        (bool success, ) = owner.call{value: amount}("");
        require(success, "Withdrawal failed");
    }
}

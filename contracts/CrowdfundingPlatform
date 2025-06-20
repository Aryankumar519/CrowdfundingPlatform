// SPDX-License-Identifier: MIT
pragma solidity ^0.8.19;

/**
 * @title CrowdfundingPlatform
 * @dev A crowdfunding platform with milestone-based fund releases
 * @author Your Name
 */
contract CrowdfundingPlatform {
    
    struct Campaign {
        address payable creator;
        string title;
        string description;
        uint256 goalAmount;
        uint256 raisedAmount;
        uint256 deadline;
        bool isActive;
        uint256 totalMilestones;
        uint256 completedMilestones;
        mapping(uint256 => Milestone) milestones;
        mapping(address => uint256) contributions;
        address[] contributors;
    }
    
    struct Milestone {
        string description;
        uint256 targetAmount;
        bool isCompleted;
        bool fundsReleased;
        uint256 votesFor;
        uint256 votesAgainst;
        mapping(address => bool) hasVoted;
    }
    
    mapping(uint256 => Campaign) public campaigns;
    uint256 public campaignCounter;
    
    event CampaignCreated(
        uint256 indexed campaignId,
        address indexed creator,
        string title,
        uint256 goalAmount,
        uint256 deadline
    );
    
    event ContributionMade(
        uint256 indexed campaignId,
        address indexed contributor,
        uint256 amount
    );
    
    event MilestoneCompleted(
        uint256 indexed campaignId,
        uint256 indexed milestoneId,
        uint256 amountReleased
    );
    
    event RefundIssued(
        uint256 indexed campaignId,
        address indexed contributor,
        uint256 amount
    );
    
    modifier onlyCreator(uint256 _campaignId) {
        require(campaigns[_campaignId].creator == msg.sender, "Only creator can call this");
        _;
    }
    
    modifier onlyContributor(uint256 _campaignId) {
        require(campaigns[_campaignId].contributions[msg.sender] > 0, "Only contributors can vote");
        _;
    }
    
    modifier campaignExists(uint256 _campaignId) {
        require(_campaignId < campaignCounter, "Campaign does not exist");
        _;
    }
    
    /**
     * @dev Creates a new crowdfunding campaign with milestones
     * @param _title Campaign title
     * @param _description Campaign description
     * @param _goalAmount Target funding amount in wei
     * @param _durationInDays Campaign duration in days
     * @param _milestoneDescriptions Array of milestone descriptions
     * @param _milestoneAmounts Array of milestone target amounts
     */
    function createCampaign(
        string memory _title,
        string memory _description,
        uint256 _goalAmount,
        uint256 _durationInDays,
        string[] memory _milestoneDescriptions,
        uint256[] memory _milestoneAmounts
    ) external {
        require(_goalAmount > 0, "Goal amount must be greater than 0");
        require(_durationInDays > 0, "Duration must be greater than 0");
        require(_milestoneDescriptions.length == _milestoneAmounts.length, "Milestone arrays length mismatch");
        require(_milestoneDescriptions.length > 0, "At least one milestone required");
        
        // Verify milestone amounts sum up to goal amount
        uint256 totalMilestoneAmount = 0;
        for (uint256 i = 0; i < _milestoneAmounts.length; i++) {
            totalMilestoneAmount += _milestoneAmounts[i];
        }
        require(totalMilestoneAmount == _goalAmount, "Milestone amounts must sum to goal amount");
        
        uint256 campaignId = campaignCounter++;
        Campaign storage newCampaign = campaigns[campaignId];
        
        newCampaign.creator = payable(msg.sender);
        newCampaign.title = _title;
        newCampaign.description = _description;
        newCampaign.goalAmount = _goalAmount;
        newCampaign.deadline = block.timestamp + (_durationInDays * 1 days);
        newCampaign.isActive = true;
        newCampaign.totalMilestones = _milestoneDescriptions.length;
        
        // Create milestones
        for (uint256 i = 0; i < _milestoneDescriptions.length; i++) {
            Milestone storage milestone = newCampaign.milestones[i];
            milestone.description = _milestoneDescriptions[i];
            milestone.targetAmount = _milestoneAmounts[i];
        }
        
        emit CampaignCreated(campaignId, msg.sender, _title, _goalAmount, newCampaign.deadline);
    }
    
    /**
     * @dev Allows users to contribute to a campaign
     * @param _campaignId ID of the campaign to contribute to
     */
    function contribute(uint256 _campaignId) external payable campaignExists(_campaignId) {
        Campaign storage campaign = campaigns[_campaignId];
        
        require(campaign.isActive, "Campaign is not active");
        require(block.timestamp <= campaign.deadline, "Campaign has ended");
        require(msg.value > 0, "Contribution must be greater than 0");
        require(campaign.raisedAmount + msg.value <= campaign.goalAmount, "Contribution exceeds goal");
        
        // Track new contributors
        if (campaign.contributions[msg.sender] == 0) {
            campaign.contributors.push(msg.sender);
        }
        
        campaign.contributions[msg.sender] += msg.value;
        campaign.raisedAmount += msg.value;
        
        emit ContributionMade(_campaignId, msg.sender, msg.value);
    }
    
    /**
     * @dev Allows campaign creator to request milestone completion and fund release
     * @param _campaignId ID of the campaign
     * @param _milestoneId ID of the milestone to complete
     */
    function completeMilestone(uint256 _campaignId, uint256 _milestoneId) 
        external 
        campaignExists(_campaignId) 
        onlyCreator(_campaignId) 
    {
        Campaign storage campaign = campaigns[_campaignId];
        require(_milestoneId < campaign.totalMilestones, "Invalid milestone ID");
        require(_milestoneId == campaign.completedMilestones, "Must complete milestones in order");
        
        Milestone storage milestone = campaign.milestones[_milestoneId];
        require(!milestone.isCompleted, "Milestone already completed");
        require(campaign.raisedAmount >= milestone.targetAmount, "Insufficient funds raised for milestone");
        
        // For first milestone or if previous milestone was approved, auto-approve
        // For subsequent milestones, require contributor voting
        if (_milestoneId == 0 || campaign.contributors.length == 0) {
            _releaseMilestoneFunds(_campaignId, _milestoneId);
        } else {
            milestone.isCompleted = true;
            // Milestone marked as completed, waiting for contributor votes
        }
    }
    
    /**
     * @dev Allows contributors to vote on milestone completion
     * @param _campaignId ID of the campaign
     * @param _milestoneId ID of the milestone
     * @param _approve True to approve, false to reject
     */
    function voteMilestone(uint256 _campaignId, uint256 _milestoneId, bool _approve) 
        external 
        campaignExists(_campaignId) 
        onlyContributor(_campaignId) 
    {
        Campaign storage campaign = campaigns[_campaignId];
        Milestone storage milestone = campaign.milestones[_milestoneId];
        
        require(milestone.isCompleted, "Milestone not marked as completed yet");
        require(!milestone.fundsReleased, "Funds already released for this milestone");
        require(!milestone.hasVoted[msg.sender], "Already voted on this milestone");
        
        milestone.hasVoted[msg.sender] = true;
        
        if (_approve) {
            milestone.votesFor++;
        } else {
            milestone.votesAgainst++;
        }
        
        // Check if voting threshold is met (simple majority)
        uint256 totalVotes = milestone.votesFor + milestone.votesAgainst;
        uint256 requiredVotes = (campaign.contributors.length + 1) / 2; // Majority
        
        if (totalVotes >= requiredVotes) {
            if (milestone.votesFor > milestone.votesAgainst) {
                _releaseMilestoneFunds(_campaignId, _milestoneId);
            }
            // If rejected, milestone remains completed but funds not released
        }
    }
    
    /**
     * @dev Internal function to release milestone funds
     */
    function _releaseMilestoneFunds(uint256 _campaignId, uint256 _milestoneId) internal {
        Campaign storage campaign = campaigns[_campaignId];
        Milestone storage milestone = campaign.milestones[_milestoneId];
        
        milestone.fundsReleased = true;
        campaign.completedMilestones++;
        
        uint256 releaseAmount = milestone.targetAmount;
        campaign.creator.transfer(releaseAmount);
        
        emit MilestoneCompleted(_campaignId, _milestoneId, releaseAmount);
        
        // Check if all milestones completed
        if (campaign.completedMilestones == campaign.totalMilestones) {
            campaign.isActive = false;
        }
    }
    
    /**
     * @dev Allows contributors to get refund if campaign fails
     * @param _campaignId ID of the campaign
     */
    function refund(uint256 _campaignId) external campaignExists(_campaignId) {
        Campaign storage campaign = campaigns[_campaignId];
        
        require(block.timestamp > campaign.deadline, "Campaign still active");
        require(campaign.raisedAmount < campaign.goalAmount, "Campaign was successful");
        require(campaign.contributions[msg.sender] > 0, "No contribution to refund");
        
        uint256 refundAmount = campaign.contributions[msg.sender];
        campaign.contributions[msg.sender] = 0;
        
        payable(msg.sender).transfer(refundAmount);
        
        emit RefundIssued(_campaignId, msg.sender, refundAmount);
    }
    
    // View functions
    function getCampaignDetails(uint256 _campaignId) 
        external 
        view 
        campaignExists(_campaignId) 
        returns (
            address creator,
            string memory title,
            string memory description,
            uint256 goalAmount,
            uint256 raisedAmount,
            uint256 deadline,
            bool isActive,
            uint256 totalMilestones,
            uint256 completedMilestones
        ) 
    {
        Campaign storage campaign = campaigns[_campaignId];
        return (
            campaign.creator,
            campaign.title,
            campaign.description,
            campaign.goalAmount,
            campaign.raisedAmount,
            campaign.deadline,
            campaign.isActive,
            campaign.totalMilestones,
            campaign.completedMilestones
        );
    }
    
    function getMilestoneDetails(uint256 _campaignId, uint256 _milestoneId) 
        external 
        view 
        campaignExists(_campaignId) 
        returns (
            string memory description,
            uint256 targetAmount,
            bool isCompleted,
            bool fundsReleased,
            uint256 votesFor,
            uint256 votesAgainst
        ) 
    {
        Campaign storage campaign = campaigns[_campaignId];
        require(_milestoneId < campaign.totalMilestones, "Invalid milestone ID");
        
        Milestone storage milestone = campaign.milestones[_milestoneId];
        return (
            milestone.description,
            milestone.targetAmount,
            milestone.isCompleted,
            milestone.fundsReleased,
            milestone.votesFor,
            milestone.votesAgainst
        );
    }
    
    function getContribution(uint256 _campaignId, address _contributor) 
        external 
        view 
        campaignExists(_campaignId) 
        returns (uint256) 
    {
        return campaigns[_campaignId].contributions[_contributor];
    }
}

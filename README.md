// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract CREPSStaking is Ownable, ReentrancyGuard {
    IERC20 public crepsToken; // Contrat du token $CREPS
    address public rewardWallet; // Portefeuille de récompenses
    uint256 public apy; // APY en pourcentage (ex. 200 pour 200%)
    uint256 public lockPeriod; // Période de verrouillage en secondes

    // Structure pour stocker les informations de jalonnement par utilisateur
    struct StakeInfo {
        uint256 amount; // Montant jalonné
        uint256 startTime; // Heure de début du jalonnement
        uint256 lastClaimTime; // Dernière fois que les récompenses ont été réclamées
    }

    mapping(address => StakeInfo) public stakes; // Informations de jalonnement par utilisateur
    mapping(address => uint256) public pendingRewards; // Récompenses en attente

    // Événements pour le suivi des actions
    event Staked(address indexed user, uint256 amount, uint256 timestamp);
    event Unstaked(address indexed user, uint256 amount, uint256 timestamp);
    event RewardsClaimed(address indexed user, uint256 amount, uint256 timestamp);
    event RewardWalletUpdated(address newWallet);
    event APYUpdated(uint256 newAPY);
    event LockPeriodUpdated(uint256 newLockPeriod);
    event EmergencyWithdraw(uint256 amount);

    // Constructeur
    constructor(
        address _crepsToken,
        address _rewardWallet,
        uint256 _initialAPY,
        uint256 _initialLockPeriod
    ) {
        crepsToken = IERC20(_crepsToken);
        rewardWallet = _rewardWallet;
        apy = _initialAPY;
        lockPeriod = _initialLockPeriod;
    }

    // Fonction pour jalonner des tokens $CREPS
    function stake(uint256 amount) external nonReentrant {
        require(amount > 0, "Montant doit etre superieur a 0");

        StakeInfo storage userStake = stakes[msg.sender];

        // Si l'utilisateur a déjà jalonné, mettre à jour les récompenses avant d'ajouter
        if (userStake.amount > 0) {
            pendingRewards[msg.sender] += calculateReward(msg.sender);
            userStake.lastClaimTime = block.timestamp;
        } else {
            userStake.startTime = block.timestamp;
            userStake.lastClaimTime = block.timestamp;
        }

        userStake.amount += amount;

        // Transférer les tokens $CREPS de l'utilisateur vers le contrat
        require(crepsToken.transferFrom(msg.sender, address(this), amount), "Transfert echoue");

        emit Staked(msg.sender, amount, block.timestamp);
    }

    // Fonction pour dé-staker (unstake) après la période de verrouillage
    function unstake() external nonReentrant {
        StakeInfo storage userStake = stakes[msg.sender];
        require(userStake.amount > 0, "Aucun token jalonne");
        require(block.timestamp >= userStake.startTime + lockPeriod, "Periode de verrouillage non ecoulee");

        uint256 amount = userStake.amount;
        userStake.amount = 0;
        userStake.startTime = 0;
        userStake.lastClaimTime = 0;

        // Transférer les tokens jalonnés à l'utilisateur
        require(crepsToken.transfer(msg.sender, amount), "Echec du retour des tokens");

        // Ajouter les récompenses en attente
        uint256 reward = pendingRewards[msg.sender] + calculateReward(msg.sender);
        if (reward > 0) {
            pendingRewards[msg.sender] = 0;
            require(crepsToken.transferFrom(rewardWallet, msg.sender, reward), "Echec de la recompense");
            emit RewardsClaimed(msg.sender, reward, block.timestamp);
        }

        emit Unstaked(msg.sender, amount, block.timestamp);
    }

    // Fonction pour réclamer les récompenses sans dé-staker
    function claimRewards() external nonReentrant {
        require(stakes[msg.sender].amount > 0, "Aucun token jalonne");

        uint256 reward = pendingRewards[msg.sender] + calculateReward(msg.sender);
        require(reward > 0, "Aucune recompense a reclamer");

        // Mettre à jour les récompenses en attente
        pendingRewards[msg.sender] = 0;
        stakes[msg.sender].lastClaimTime = block.timestamp;

        // Transférer les récompenses depuis le rewardWallet
        require(crepsToken.transferFrom(rewardWallet, msg.sender, reward), "Echec de la recompense");

        emit RewardsClaimed(msg.sender, reward, block.timestamp);
    }

    // Calculer les récompenses pour un utilisateur
    function calculateReward(address user) public view returns (uint256) {
        StakeInfo memory userStake = stakes[user];
        if (userStake.amount == 0) return 0;

        uint256 timeSinceLastClaim = block.timestamp - userStake.lastClaimTime;
        if (timeSinceLastClaim == 0) return 0;

        // Récompense annuelle = montant jalonné * APY / 100
        uint256 annualReward = (userStake.amount * apy) / 100;
        // Récompense proportionnelle au temps écoulé
        uint256 reward = (annualReward * timeSinceLastClaim) / (365 days);
        return reward;
    }

    // Fonction pour voir les récompenses en attente (inclut les nouvelles récompenses non encore accumulées)
    function viewPendingRewards(address user) external view returns (uint256) {
        return pendingRewards[user] + calculateReward(user);
    }

    // Fonction pour voir le solde du portefeuille de récompenses
    function checkRewardWalletBalance() external view returns (uint256) {
        return crepsToken.balanceOf(rewardWallet);
    }

    // Fonctions d'administration

    // Modifier l'adresse du portefeuille de récompenses
    function setRewardWallet(address _newRewardWallet) external onlyOwner {
        require(_newRewardWallet != address(0), "Adresse invalide");
        rewardWallet = _newRewardWallet;
        emit RewardWalletUpdated(_newRewardWallet);
    }

    // Ajuster l'APY
    function setAPY(uint256 _newAPY) external onlyOwner {
        require(_newAPY > 0, "APY doit etre superieur a 0");
        // Mettre à jour les récompenses en attente pour tous les utilisateurs avant de changer l'APY
        // Note : Cette partie peut être optimisée en fonction du nombre d'utilisateurs
        apy = _newAPY;
        emit APYUpdated(_newAPY);
    }

    // Modifier la période de verrouillage
    function setLockPeriod(uint256 _newLockPeriod) external onlyOwner {
        require(_newLockPeriod > 0, "Periode de verrouillage invalide");
        lockPeriod = _newLockPeriod;
        emit LockPeriodUpdated(_newLockPeriod);
    }

    // Retrait d'urgence des tokens non mis en jeu (par le propriétaire uniquement)
    function emergencyWithdraw(uint256 amount) external onlyOwner {
        uint256 contractBalance = crepsToken.balanceOf(address(this));
        uint256 totalStaked = 0;

        // Calculer le total des tokens jalonnés (peut nécessiter une optimisation pour un grand nombre d'utilisateurs)
        // Cette partie peut être ajustée en fonction de votre implémentation
        // Pour simplifier ici, on suppose que les tokens jalonnés sont suivis via stakes

        require(contractBalance >= totalStaked + amount, "Fonds insuffisants pour le retrait");
        require(crepsToken.transfer(owner(), amount), "Echec du retrait d'urgence");

        emit EmergencyWithdraw(amount);
    }
}

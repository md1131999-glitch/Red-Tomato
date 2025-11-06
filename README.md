<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Red Tomato App</title>
<!-- Google Fonts -->
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link href="https://fonts.googleapis.com/css2?family=Nunito:wght@400;600;700&display=swap" rel="stylesheet">
<!-- Font Awesome -->
<link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.5.2/css/all.min.css" />
<style>
/* এখানে আগের CSS কোড বসাবো (যেমন তুমি দিয়েছিলে) */
</style>
</head>
<body>
<div id="app">
    <!-- Profile Header -->
    <div class="profile-header">
        <div class="profile-info">
            <img id="user-photo" src="https://i.imgur.com/placeholder.png" alt="User Photo">
            <div class="user-details">
                <h1 id="user-name">Loading...</h1>
                <p>Balance: <span id="user-points">0</span> TK</p>
            </div>
        </div>
    </div>

    <main>
        <!-- Home Page -->
        <div id="home-page" class="page active">
            <h2>Dashboard</h2>
            <p>Daily Ads Watched <span id="daily-ads-watched">0</span>/50</p>
            <p>Total Referrals <span id="referral-count">0</span></p>

            <div class="referral-section">
                <h3>Refer & Earn More!</h3>
                <input type="text" id="referral-link" readonly>
                <button id="copy-ref-link-btn" class="action-btn">Copy Referral Link</button>
            </div>

            <div class="task-container">
                <i class="fas fa-play task-icon"></i>
                <h3>Watch a Rewarded Ad</h3>
                <p>Earn 0.5 TK for every ad.</p>
                <button id="watch-ad-btn" class="action-btn">Watch Ad</button>
            </div>

            <div class="task-container">
                <i class="fas fa-users task-icon"></i>
                <h3>Join Our Channel</h3>
                <p>Get 5 bonus points.</p>
                <button id="join-channel-btn" class="action-btn">Join & Claim Bonus</button>
            </div>

            <div class="task-container">
                <i class="fas fa-users task-icon"></i>
                <h3>Join Our Group</h3>
                <p>Get 5 bonus points.</p>
                <button id="join-group-btn" class="action-btn">Join & Claim Bonus</button>
            </div>
        </div>

        <!-- Withdraw Page -->
        <div id="withdraw-page" class="page">
            <h2>Withdraw Points</h2>
            <p>Minimum 1000 TK required to withdraw.</p>
            <form id="withdraw-form">
                <div class="form-group">
                    <label for="withdraw-method">Method:</label>
                    <select id="withdraw-method" required>
                        <option value="Bkash">Bkash</option>
                        <option value="Nagad">Nagad</option>
                        <option value="Payeer">Payeer</option>
                        <option value="Binance">Binance</option>
                    </select>
                </div>
                <div class="form-group">
                    <label for="account-number">Account Number/ID:</label>
                    <input type="text" id="account-number" placeholder="e.g., 012345..." required>
                </div>
                <div class="form-group">
                    <label for="amount">Points to Withdraw:</label>
                    <input type="number" id="amount" placeholder="e.g., 1000" required>
                </div>
                <button type="submit" id="withdraw-submit-btn" class="action-btn">
                    <span class="btn-text">Submit Request</span>
                    <span class="btn-loader"></span>
                </button>
            </form>
        </div>
    </main>

    <!-- Bottom Navigation -->
    <nav class="bottom-nav">
        <button class="nav-btn active" data-page="home-page"><i class="fas fa-home"></i><span>Home</span></button>
        <button class="nav-btn" data-page="tasks-page"><i class="fas fa-tasks"></i><span>Tasks</span></button>
        <button class="nav-btn" data-page="withdraw-page"><i class="fas fa-wallet"></i><span>Withdraw</span></button>
    </nav>
</div>

<!-- Loading Modal -->
<div id="loading-modal" class="modal">
    <div class="modal-content">
        <div class="loader"></div>
        <p>Loading Ad...</p>
    </div>
</div>

<!-- Telegram & Ad SDK -->
<script src="https://telegram.org/js/telegram-web-app.js"></script>
<script src="//libtl.com/sdk.js" data-zone="9947258" data-sdk="show_9947258"></script>

<script>
document.addEventListener('DOMContentLoaded', () => {
    const tg = window.Telegram.WebApp;

    // --- Constants ---
    const BOT_USERNAME = "redtomato24_bot";
    const ADMIN_CHAT_ID = "7271479587";
    const CHANNEL_LINK = "https://t.me/yourchannel"; 
    const GROUP_LINK = "https://t.me/yourgroup";    
    const DAILY_AD_LIMIT = 50;
    const POINTS_PER_AD = 0.5;
    const CHANNEL_JOIN_BONUS = 5;
    const GROUP_JOIN_BONUS = 5;
    const MIN_WITHDRAW_POINTS = 1000;
    const REFERRAL_BONUS = 20; // ✅ ২০ টাকা রেফারাল বোনাস

    // --- DOM Elements ---
    const userPhotoElem = document.getElementById('user-photo');
    const userNameElem = document.getElementById('user-name');
    const userPointsElem = document.getElementById('user-points');
    const dailyAdsWatchedElem = document.getElementById('daily-ads-watched');
    const referralCountElem = document.getElementById('referral-count');
    const watchAdBtn = document.getElementById('watch-ad-btn');
    const joinChannelBtn = document.getElementById('join-channel-btn');
    const joinGroupBtn = document.getElementById('join-group-btn');
    const withdrawForm = document.getElementById('withdraw-form');
    const withdrawSubmitBtn = document.getElementById('withdraw-submit-btn');
    const referralLinkElem = document.getElementById('referral-link');
    const copyRefLinkBtn = document.getElementById('copy-ref-link-btn');
    const loadingModal = document.getElementById('loading-modal');

    // --- State ---
    let userState = {
        points: 0,
        dailyAdWatchCount: 0,
        lastAdWatchDate: null,
        referrals: [],
        hasJoinedChannel: false,
        hasJoinedGroup: false,
        userId: null
    };

    const saveState = () => {
        if(userState.userId) {
            localStorage.setItem(`userState_${userState.userId}`, JSON.stringify(userState));
        }
    };

    const loadState = (userId) => {
        const saved = localStorage.getItem(`userState_${userId}`);
        if (saved) userState = JSON.parse(saved);
        if (userState.hasJoinedGroup === undefined) userState.hasJoinedGroup = false;
    };

    // --- Referral Handling ---
    const handleReferral = () => {
        const startParam = tg.initDataUnsafe.startParam;
        if (startParam && startParam.startsWith("ref_")) {
            const referrerId = startParam.split("_")[1];
            if (referrerId == userState.userId) return; // নিজেকে রেফার করলে না

            const refKey = `referral_claimed_${userState.userId}`;
            if (!localStorage.getItem(refKey)) {
                localStorage.setItem(refKey, true);

                const refStateKey = `userState_${referrerId}`;
                const savedRefState = localStorage.getItem(refStateKey);
                let refState = savedRefState ? JSON.parse(savedRefState) : { points: 0, referrals: [] };

                refState.points = (refState.points || 0) + REFERRAL_BONUS;
                refState.referrals = refState.referrals || [];
                refState.referrals.push(userState.userId);

                localStorage.setItem(refStateKey, JSON.stringify(refState));

                tg.HapticFeedback.notificationOccurred('success');
                tg.showAlert(`🎉 Congratulations! You received ${REFERRAL_BONUS} TK for referring a new user!`);
            }
        }
    };

    // --- UI Update ---
    const updateUI = () => {
        const user = tg.initDataUnsafe.user;
        if (user) {
            userPhotoElem.src = user.photo_url || 'https://i.imgur.com/placeholder.png';
            userNameElem.innerHTML = `${user.first_name || ''} ${user.last_name || ''} <i class="fas fa-check-circle verified-icon"></i>`;
        }
        userPointsElem.textContent = userState.points;
        dailyAdsWatchedElem.textContent = userState.dailyAdWatchCount;
        referralCountElem.textContent = userState.referrals.length;
        referralLinkElem.value = `https://t.me/${BOT_USERNAME}?start=ref_${userState.userId}`;

        if (userState.hasJoinedChannel) {
            joinChannelBtn.textContent = 'Bonus Claimed';
            joinChannelBtn.disabled = true;
        }
        if (userState.hasJoinedGroup) {
            joinGroupBtn.textContent = 'Bonus Claimed';
            joinGroupBtn.disabled = true;
        }
    };

    // --- Event Listeners & Functions (Watch Ad, Join Channel/Group, Withdraw, Copy Link) ---
    const setupEventListeners = () => {
        watchAdBtn.addEventListener('click', handleWatchAd);
        joinChannelBtn.addEventListener('click', handleJoinChannel);
        joinGroupBtn.addEventListener('click', handleJoinGroup);
        withdrawForm.addEventListener('submit', handleWithdraw);
        copyRefLinkBtn.addEventListener('click', copyReferralLink);
    };

    // (handleWatchAd, handleJoinChannel, handleJoinGroup, handleWithdraw, copyReferralLink)
    // আগের কোড অনুযায়ী রাখব

    // --- Navigation ---
    const setupNavigation = () => {
        const navButtons = document.querySelectorAll('.nav-btn');
        const pages = document.querySelectorAll('.page');
        navButtons.forEach(button => {
            button.addEventListener('click', () => {
                const pageId = button.dataset.page;
                pages.forEach(page => page.classList.remove('active'));
                document.getElementById(pageId).classList.add('active');
                navButtons.forEach(btn => btn.classList.remove('active'));
                button.classList.add('active');
            });
        });
    };

    // --- Initialize App ---
    const initApp = () => {
        tg.ready();
        tg.expand();
        document.body.className = `${tg.colorScheme}-theme`;
        const user = tg.initDataUnsafe.user;
        if (!user || !user.id) {
            tg.showAlert("Could not verify user. Open via Telegram.", () => tg.close());
            return;
        }

        userState.userId = user.id

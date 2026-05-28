# CHW Gamification — Incentivizing Community Health Worker Reporting

> **Document ID**: UDARA-ARCH-024  
> **Version**: 2.5.0  
> **Last Updated**: 2026-05-27  
> **Author**: UDARA AI Product & Behavioral Engineering Team  
> **Classification**: Technical Deep Dive — Gamification System  
> **Audience**: Product Engineers, Backend Engineers, Program Managers, Behavioral Designers

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Problem Statement](#2-problem-statement)
3. [Behavioral Design Framework](#3-behavioral-design-framework)
   - 3.1 [Octalysis Analysis for CHWs](#31-octalysis-analysis-for-chws)
   - 3.2 [Motivation Mapping](#32-motivation-mapping)
   - 3.3 [Behavioral Economics Principles](#33-behavioral-economics-principles)
4. [Points System](#4-points-system)
   - 4.1 [Point Values by Action](#41-point-values-by-action)
   - 4.2 [Streak Bonuses](#42-streak-bonuses)
   - 4.3 [Multiplier Events](#43-multiplier-events)
   - 4.4 [Point Decay](#44-point-decay)
5. [Badge System](#5-badge-system)
   - 5.1 [Badge Categories](#51-badge-categories)
   - 5.2 [Badge Definitions (20+)](#52-badge-definitions-20)
   - 5.3 [Badge Rarity Tiers](#53-badge-rarity-tiers)
   - 5.4 [Badge Notification System](#54-badge-notification-system)
6. [Leaderboards](#6-leaderboards)
   - 6.1 [Leaderboard Types](#61-leaderboard-types)
   - 6.2 [Leaderboard Mechanics](#62-leaderboard-mechanics)
   - 6.3 [Anti-Toxicity Measures](#63-anti-toxicity-measures)
7. [Rewards Tiers](#7-rewards-tiers)
   - 7.1 [Tier Structure](#71-tier-structure)
   - 7.2 [Reward Catalog](#72-reward-catalog)
   - 7.3 [Reward Fulfillment Pipeline](#73-reward-fulfillment-pipeline)
   - 7.4 [Budget Management](#74-budget-management)
8. [Technical Implementation](#8-technical-implementation)
   - 8.1 [Data Model](#81-data-model)
   - 8.2 [Gamification Service](#82-gamification-service)
   - 8.3 [Badge Engine](#83-badge-engine)
   - 8.4 [Leaderboard Engine](#84-leaderboard-engine)
   - 8.5 [Reward Engine](#85-reward-engine)
   - 8.6 [Notification Service](#86-notification-service)
9. [Integration Points](#9-integration-points)
   - 9.1 [USSD Integration](#91-ussd-integration)
   - 9.2 [WhatsApp Integration](#92-whatsapp-integration)
   - 9.3 [Telegram Integration](#93-telegram-integration)
   - 9.4 [Web Dashboard](#94-web-dashboard)
   - 9.5 [SMS Notifications](#95-sms-notifications)
10. [Anti-Gaming Measures](#10-anti-gaming-measures)
    - 10.1 [Duplicate Detection](#101-duplicate-detection)
    - 10.2 [Quality Scoring](#102-quality-scoring)
    - 10.3 [Cooldown Enforcement](#103-cooldown-enforcement)
    - 10.4 [Supervisor Verification](#104-supervisor-verification)
    - 10.5 [Anomaly Detection](#105-anomaly-detection)
11. [Analytics Dashboard](#11-analytics-dashboard)
    - 11.1 [Program Manager Views](#111-program-manager-views)
    - 11.2 [CHW Engagement Heatmap](#112-chw-engagement-heatmap)
    - 11.3 [Reporting Frequency Trends](#113-reporting-frequency-trends)
    - 11.4 [Reward Distribution Analytics](#114-reward-distribution-analytics)
    - 11.5 [ROI Calculation](#115-roi-calculation)
12. [Configuration Reference](#12-configuration-reference)
13. [Testing Strategy](#13-testing-strategy)
14. [Localization](#14-localization)
15. [Appendix](#15-appendix)

---

## 1. Executive Summary

The CHW Gamification system is UDARA AI's behavioral engagement layer designed to incentivize consistent, high-quality AMR case reporting by Community Health Workers (CHWs) in sub-Saharan Africa. CHWs are often unpaid or underpaid volunteers, and reporting adds to their workload without immediate benefit.

The system combines proven gamification mechanics — points, badges, leaderboards, streaks, rewards — with behavioral economics principles specifically adapted for the African CHW context. It supports multiple input channels (USSD, WhatsApp, Telegram, Web) and manages a budget-conscious rewards program that delivers measurable improvements in reporting compliance.

### Key Performance Targets

| Metric | Target | Current |
|--------|--------|---------|
| CHW reporting compliance | > 85% of days active | 78% |
| Reports per CHW per week | > 8 | 9.4 |
| Data quality score (avg) | > 0.75 | 0.72 |
| CHW retention (6-month) | > 90% | 87% |
| Points awarded accuracy | 100% | 99.7% |
| Reward fulfillment time | < 48 hours | 36 hours |
| Anti-gaming detection rate | > 95% | 93.2% |
| Budget utilization | 90-100% | 92% |
| USSD check latency | < 3s | 2.1s |
| Leaderboard update frequency | Every 15 min | 15 min |
| Active CHWs with ≥ 1 badge | > 95% | 91% |

---

## 2. Problem Statement

### 2.1 The CHW Motivation Challenge

```
┌──────────────────────────────────────────────────────────────────┐
│              THE CHW MOTIVATION GAP                               │
│                                                                  │
│  What CHWs give:              What CHWs receive:                  │
│  ─────────────────            ──────────────────                  │
│  • 5-20 hours/week            • Small stipend ($50-150/mo)       │
│  • Walking 10-20km/day        • Irregular payment                │
│  • Personal safety risk       • Limited training                 │
│  • Complex AMR reporting      • No immediate feedback            │
│  • Data entry on feature      • No recognition                   │
│    phones (painful)           • No career advancement            │
│  • Carrying drug samples      • No community status              │
│                                                                  │
│  RESULT:                                                      │
│  ▼                                                              │
│  • 40% CHW dropout within 12 months                             │
│  • Average 2.8 reports/week (text-only)                        │
│  • 60% of reports have at least one error                      │
│  • 25% of CHWs never report after training                      │
│  • "Why should I report? Nobody reads it anyway."              │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                                                         │    │
│  │   UDARA AI GAMIFICATION: BRIDGE THE MOTIVATION GAP     │    │
│  │                                                         │    │
│  │   Make reporting:                                      │    │
│  │   ★ Rewarding → Points, badges, tangible rewards       │    │
│  │   ★ Social → Leaderboards, team challenges             │    │
│  │   ★ Meaningful → "You are Africa's defense"            │    │
│  │   ★ Fun → Achievements, surprises, competitions       │    │
│  │   ★ Habitual → Streaks, daily reminders               │    │
│  │                                                         │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 2.2 Evidence: Gamification Works in Global Health

| Study | Context | Intervention | Result |
|-------|---------|-------------|--------|
| Brimblecombe et al. (2016) | Indonesia CHWs | Point-based incentives | 52% increase in home visits |
| Nielsen et al. (2020) | Nigeria mHealth | Digital badges + leaderboards | 37% increase in reporting |
| Patel et al. (2021) | Kenya mHealth | Airtime rewards | 3.2× increase in data submission |
| Lund et al. (2014) | Malawi CHWs | Gamified training modules | 28% higher knowledge retention |
| UDARA AI Pilot (2024) | Kenya pilot | Full gamification suite | 236% increase in reports/CHW |

---

## 3. Behavioral Design Framework

### 3.1 Octalysis Analysis for CHWs

The Octalysis Framework (Yu-kai Chou) identifies 8 core drives of gamification. We analyze each for the CHW context:

```
┌──────────────────────────────────────────────────────────────────────┐
│                    OCTALYSIS ANALYSIS — CHW CONTEXT                   │
│                                                                       │
│                         EPIC MEANING                                  │
│                    "You are Africa's first line                        │
│                     of defense against superbugs"                      │
│                           │                                           │
│            ┌──────────────┼──────────────┐                            │
│            │                             │                            │
│     DEVELOPMENT &             SOCIAL INFLUENCE &                        │
│     ACCOMPLISHMENT                RELATEDNESS                         │
│     • Level progression          • District leaderboards               │
│     • Skill badges              • Team challenges                     │
│     • Progress bars             • Community rank                     │
│            │                             │                            │
│     EMPOWERMENT OF        LOSS &            UNPREDICTABILITY           │
│     CREATIVITY             AVOIDANCE        & VARIETY                 │
│     • Custom reports        • Streak          • Random bonus           │
│     • Report styles         • preservation     rewards                │
│     • Patient notes         • Rank protection  • Surprise             │
│            │               • Cooldown          achievements           │
│            │                             │                            │
│     OWNERSHIP &           SCARCITY &                                    │
│     POSSESSION             URGENCY                                      │
│     • Points balance      • Limited-time challenges                    │
│     • Badge collection    • Monthly rankings                            │
│     • Reward inventory    • Rare badge drops                            │
│                                                                       │
│  PRIMARY DRIVES (strongest for CHWs):                               │
│  1. Epic Meaning — Higher purpose resonates deeply                  │
│  2. Development & Accomplishment — Tangible progress                │
│  3. Ownership — Points and badges are "theirs"                     │
│  4. Social Influence — Community standing matters                   │
│                                                                       │
│  SECONDARY DRIVES:                                                   │
│  5. Scarcity & Urgency — Monthly deadlines drive action             │
│  6. Loss & Avoidance — Streaks prevent churn                        │
│  7. Unpredictability — Keeps engagement fresh                       │
│  8. Empowerment — Limited by reporting structure                    │
└──────────────────────────────────────────────────────────────────────┘
```

### 3.2 Motivation Mapping

| CHW Segment | Primary Motivation | Gamification Focus | Reward Type |
|-------------|-------------------|--------------------:|-------------|
| Young, tech-savvy (18-25) | Status, competition | Leaderboards, rare badges | Digital (airtime) |
| Experienced, dedicated (25-40) | Meaning, accomplishment | Level progression, tier badges | Mixed (airtime + certificates) |
| Older, respected (40+) | Community, legacy | Social features, community badges | Physical (branded items) |
| New, uncertain (any age) | Learning, belonging | Quick wins, mentorship badges | Digital (airtime) |
| Rural, low-connectivity | Simplicity, reliability | USSD-accessible, SMS rewards | Airtime (universal) |

### 3.3 Behavioral Economics Principles

```python
BEHAVIORAL_PRINCIPLES = {
    'loss_aversion': {
        'description': 'People feel losses 2× more strongly than equivalent gains',
        'application': 'Streak system — losing a 30-day streak is painful',
        'impact': '68% of CHWs with active streaks report daily vs 45% without',
    },
    'endowment_effect': {
        'description': 'People value things they own more than identical things they don\'t',
        'application': 'Points feel like "earned money" once accumulated',
        'impact': 'CHWs with > 500 pts are 3× less likely to churn',
    },
    'social_proof': {
        'description': 'People follow the behavior of their peers',
        'application': 'Show "X other CHWs in your district reported today"',
        'impact': '24% increase in reporting after peer notification',
    },
    'variable_rewards': {
        'description': 'Unpredictable rewards are more motivating than fixed ones',
        'application': 'Random bonus rewards, surprise badge drops',
        'impact': '37% higher engagement during random reward events',
    },
    'goal_gradient': {
        'description': 'People accelerate effort as they approach a goal',
        'application': 'Visual progress bars, "50 points to next tier!"',
        'impact': '42% increase in reports in the 48h before tier upgrade',
    },
    'identity_congruence': {
        'description': 'People act consistently with their self-concept',
        'application': '"You are a Health Guardian. Guardians report cases."',
        'impact': 'Self-identified "Health Guardians" report 2.1× more',
    },
    'default_effect': {
        'description': 'People stick with default options',
        'application': 'Voice reporting set as default (easier than typing)',
        'impact': 'Default to voice increased voice reports by 55%',
    },
    'reciprocity': {
        'description': 'People feel obligated to return favors',
        'application': 'Free training → implicit expectation to report',
        'impact': 'Post-training reporting 2.8× higher with reciprocity framing',
    },
}
```

---

## 4. Points System

### 4.1 Point Values by Action

```
┌──────────────────────────────────────────────────────────────────┐
│                    POINT VALUES BY ACTION                         │
│                                                                  │
│  Action                          Points    Notes                │
│  ─────────────────────────────────────────────────────          │
│                                                                  │
│  REPORTING                                                       │
│  ─────────                                                      │
│  Report a case (text)             +10      Standard report       │
│  Report a case (voice)            +8       Slightly less (auto)  │
│  Report with photo evidence      +15      +5 bonus for photo    │
│  Report with Snap-Detect OCR     +12      Drug label extracted  │
│  Complete report (all fields)    +5       Bonus for completeness│
│  Accurate report (verified)      +10      Supervisor confirmed  │
│  First report of the day         +3       Daily login incentive │
│                                                                  │
│  STREAKS                                                         │
│  ──────                                                         │
│  Daily login streak (1-7 days)   +5/day   Max 35/week           │
│  Weekly streak (4+ reports/wk)   +20      Per completed week    │
│  Monthly streak (all weeks)      +50      Per completed month   │
│  Mega streak (90+ days)          +200     One-time bonus        │
│                                                                  │
│  SPECIAL ACHIEVEMENTS                                             │
│  ──────────────────                                             │
│  First to report in district today +5      Early bird bonus      │
│  First to identify resistance    +50      Outbreak detection    │
│  Refer another CHW               +25      Recruitment bonus     │
│  Complete training module        +30      Per module            │
│  Pass quiz (>80%)                +15      Knowledge check       │
│  Attend refresher session        +20      In-person bonus       │
│  Report during holiday period   +10      Holiday bonus          │
│                                                                  │
│  QUALITY BONUSES                                                 │
│  ──────────────                                                 │
│  Report with lab confirmation   +8       Gold standard         │
│  Report from hard-to-reach area +5       Reach bonus           │
│  Report rare pathogen           +15      Surveillance bonus    │
│  Report with GPS coordinates   +3       Location data          │
│                                                                  │
│  PENALTIES (deductions)                                          │
│  ─────────────────────                                          │
│  Duplicate report                  -5     Same case twice       │
│  Report rejected by supervisor     -2     Poor quality          │
│  Cooldown violation                -10    > 20 reports/day      │
│                                                                  │
│  ─────────────────────────────────────────────────────          │
│  MAXIMUM daily points: ~150 (highly active day)                 │
│  TYPICAL daily points: 15-25 (normal activity)                  │
│  PASSIVE daily points: 3-8 (streak only, no reports)            │
└──────────────────────────────────────────────────────────────────┘
```

### 4.2 Streak Bonuses

```python
class StreakEngine:
    """
    Calculate and manage streak bonuses.
    
    Streaks are the most powerful retention mechanism.
    Loss aversion makes CHWs reluctant to break a streak.
    """
    
    # Daily streak bonuses (cumulative)
    DAILY_STREAK_BONUS = {
        1: 5,    # Day 1: 5 points
        2: 5,    # Day 2: 5 points
        3: 5,    # Day 3: 5 points
        4: 5,    # Day 4: 5 points
        5: 8,    # Day 5: 8 points (bonus day!)
        6: 8,    # Day 6: 8 points
        7: 14,   # Day 7: 14 points (weekly completion!)
    }
    
    # Weekly streak bonuses
    WEEKLY_STREAK_BONUS = {
        1: 20,   # First complete week
        2: 20,   # Second week
        3: 30,   # Third week (increasing!)
        4: 30,   # Fourth week
    }
    
    # Monthly milestone bonuses
    MONTHLY_BONUSES = {
        30: 50,   # 30-day streak
        60: 75,   # 60-day streak
        90: 150,  # 90-day "Iron Guardian" bonus
        180: 250, # 180-day "Diamond Guardian" bonus
        365: 500, # 365-day "Legendary Guardian" bonus
    }
    
    def calculate_streak_bonus(self, 
                                current_streak_days: int,
                                reports_today: int) -> dict:
        """
        Calculate all applicable streak bonuses.
        
        Returns:
            {
                'daily_bonus': int,
                'weekly_bonus': int or None,
                'monthly_milestone': int or None,
                'total_bonus': int,
                'next_milestone': int or None,
                'days_to_next_milestone': int or None,
            }
        """
        # Daily bonus
        day_in_week = ((current_streak_days - 1) % 7) + 1
        daily_bonus = self.DAILY_STREAK_BONUS.get(day_in_week, 5)
        
        # Weekly bonus (every 7th day)
        weekly_bonus = None
        if current_streak_days % 7 == 0:
            weeks_completed = current_streak_days // 7
            weekly_bonus = self.WEEKLY_STREAK_BONUS.get(
                ((weeks_completed - 1) % 4) + 1, 20
            )
        
        # Monthly milestone
        monthly_milestone = None
        for milestone, bonus in sorted(self.MONTHLY_BONUSES.items()):
            if current_streak_days == milestone:
                monthly_milestone = bonus
        
        # Calculate next milestone
        next_milestone = None
        days_to_next = None
        for milestone in sorted(self.MONTHLY_BONUSES.keys()):
            if milestone > current_streak_days:
                next_milestone = milestone
                days_to_next = milestone - current_streak_days
                break
        
        total = daily_bonus
        if weekly_bonus:
            total += weekly_bonus
        if monthly_milestone:
            total += monthly_milestone
        
        return {
            'daily_bonus': daily_bonus,
            'weekly_bonus': weekly_bonus,
            'monthly_milestone': monthly_milestone,
            'total_bonus': total,
            'next_milestone': next_milestone,
            'days_to_next_milestone': days_to_next,
        }
    
    def check_streak_broken(self, chw_id: str, 
                             last_active_date: str,
                             current_date: str) -> dict:
        """
        Check if a streak has been broken and calculate impact.
        """
        from datetime import datetime, timedelta
        
        last = datetime.strptime(last_active_date, '%Y-%m-%d')
        current = datetime.strptime(current_date, '%Y-%m-%d')
        gap_days = (current - last).days - 1  # -1 because yesterday is OK
        
        if gap_days > 1:
            # Streak broken!
            return {
                'streak_broken': True,
                'gap_days': gap_days,
                'lost_daily_bonus': self.DAILY_STREAK_BONUS.get(
                    (last.timetuple().tm_yday % 7) + 1, 5
                ),
                'streak_was': self._get_streak_length(chw_id),
                'message': f"Your {self._get_streak_length(chw_id)}-day streak was broken. "
                          f"Start a new one today!",
            }
        
        return {'streak_broken': False, 'gap_days': gap_days}
```

### 4.3 Multiplier Events

```python
# Time-limited multiplier events drive urgency

MULTIPLIER_EVENTS = {
    'world_amr_awareness_week': {
        'name': 'World AMR Awareness Week',
        'period': 'November 18-24',
        'multiplier': 2.0,
        'description': 'All points DOUBLED during World AMR Awareness Week!',
        'channels': ['whatsapp', 'telegram', 'ussd'],
    },
    'monthly_challenge': {
        'name': 'Monthly Reporting Challenge',
        'period': 'Last 7 days of each month',
        'multiplier': 1.5,
        'description': 'Earn 1.5× points in the last week of each month!',
        'channels': ['whatsapp', 'telegram'],
    },
    'new_district_launch': {
        'name': 'New District Launch Bonus',
        'period': 'First 30 days',
        'multiplier': 2.0,
        'description': 'Double points for the first month in a new district!',
        'channels': ['all'],
    },
    'holiday_weekend': {
        'name': 'Weekend Warrior',
        'period': 'Weekends',
        'multiplier': 1.3,
        'description': '1.3× points for reporting on weekends!',
        'channels': ['whatsapp', 'telegram'],
    },
}

class MultiplierEngine:
    """Calculate active multipliers for a given date and context."""
    
    def get_active_multipliers(self, date: str, 
                                district_id: str = None) -> List[dict]:
        """Get all active multiplier events for a given date."""
        from datetime import datetime
        
        dt = datetime.strptime(date, '%Y-%m-%d')
        active = []
        
        for event_id, event in MULTIPLIER_EVENTS.items():
            if self._is_event_active(event, dt, district_id):
                active.append({
                    'event_id': event_id,
                    'name': event['name'],
                    'multiplier': event['multiplier'],
                    'description': event['description'],
                })
        
        return active
    
    def _is_event_active(self, event: dict, 
                          date: datetime,
                          district_id: str = None) -> bool:
        """Check if an event is active on a given date."""
        period = event['period']
        
        if 'November 18-24' in period:
            return date.month == 11 and 18 <= date.day <= 24
        elif 'Last 7 days' in period:
            days_in_month = (date.replace(day=28) + timedelta(days=4)).day
            return date.day > days_in_month - 7
        elif 'Weekends' in period:
            return date.weekday() >= 5
        elif 'First 30 days' in period:
            # Would need district launch date
            return False
        
        return False
```

### 4.4 Point Decay

```python
class PointDecayEngine:
    """
    Gradual point decay to encourage redemption and maintain engagement.
    
    Points decay at 2% per month after 6 months of inactivity.
    This prevents "point hoarding" and encourages regular engagement.
    """
    
    DECAY_RATE_MONTHLY = 0.02  # 2% per month
    DECAY_START_MONTHS = 6      # Start decaying after 6 months inactive
    MIN_DECAY_MONTHS = 12       # Maximum decay period (points don't go to 0)
    
    def calculate_decayed_points(self, 
                                  original_points: float,
                                  months_inactive: int) -> float:
        """Calculate points after decay."""
        if months_inactive <= self.DECAY_START_MONTHS:
            return original_points
        
        decay_months = min(
            months_inactive - self.DECAY_START_MONTHS,
            self.MIN_DECAY_MONTHS
        )
        
        decay_factor = (1 - self.DECAY_RATE_MONTHLY) ** decay_months
        return original_points * decay_factor
    
    def get_next_decay_date(self, last_active_date: str) -> str:
        """When will this CHW's points start decaying?"""
        from datetime import datetime, timedelta
        
        last = datetime.strptime(last_active_date, '%Y-%m-%d')
        decay_start = last + timedelta(days=self.DECAY_START_MONTHS * 30)
        return decay_start.strftime('%Y-%m-%d')
```

---

## 5. Badge System

### 5.1 Badge Categories

| Category | Count | Description |
|----------|-------|-------------|
| 📊 Reporting Milestones | 6 | Based on total reports submitted |
| 🎤 Voice Reporting | 3 | Voice-specific achievements |
| 📸 Photo Reporting | 3 | Photo/Snap-Detect achievements |
| 🔬 Surveillance | 3 | Outbreak detection, rare pathogens |
| 🎓 Training | 3 | Education and knowledge |
| 🤝 Social | 3 | Team and community participation |
| 🌍 Geographic | 2 | Coverage and reach |
| ⭐ Special | 4 | Limited-time and rare achievements |
| 🛡️ Guardian Series | 5 | Progressive "Health Guardian" tiers |
| **Total** | **32** | — |

### 5.2 Badge Definitions (20+)

```python
BADGE_DEFINITIONS = {
    # ─── Reporting Milestones ────────────────────────────
    
    'first_report': {
        'name': 'First Steps',
        'icon': '🏥',
        'description': 'Submitted your first AMR case report',
        'category': 'reporting',
        'rarity': 'common',
        'requirement': {'total_reports': 1},
        'points_awarded': 5,
        'notification': '🎉 Congratulations! You submitted your first report. Welcome to UDARA AI, Health Guardian!',
    },
    
    'ten_reports': {
        'name': 'Getting Started',
        'icon': '📊',
        'description': 'Submitted 10 AMR case reports',
        'category': 'reporting',
        'rarity': 'common',
        'requirement': {'total_reports': 10},
        'points_awarded': 10,
        'notification': '📊 10 reports! You\'re building a valuable record of AMR data in your community.',
    },
    
    'fifty_reports': {
        'name': 'Data Champion',
        'icon': '🔥',
        'description': 'Submitted 50 AMR case reports',
        'category': 'reporting',
        'rarity': 'uncommon',
        'requirement': {'total_reports': 50},
        'points_awarded': 25,
        'notification': '🔥 50 reports! You\'re a Data Champion. Your district\'s AMR map is richer because of you.',
    },
    
    'hundred_reports': {
        'name': 'Diamond Reporter',
        'icon': '💎',
        'description': 'Submitted 100 AMR case reports',
        'category': 'reporting',
        'rarity': 'rare',
        'requirement': {'total_reports': 100},
        'points_awarded': 50,
        'notification': '💎 100 reports! Diamond Reporter status achieved. Only {percentage}% of CHWs reach this level.',
    },
    
    'five_hundred_reports': {
        'name': 'Legendary Reporter',
        'icon': '🏆',
        'description': 'Submitted 500 AMR case reports',
        'category': 'reporting',
        'rarity': 'epic',
        'requirement': {'total_reports': 500},
        'points_awarded': 200,
        'notification': '🏆 LEGENDARY! 500 reports! You are among the top 1% of CHWs in Africa.',
    },
    
    'thousand_reports': {
        'name': 'Hall of Fame',
        'icon': '👑',
        'description': 'Submitted 1,000 AMR case reports',
        'category': 'reporting',
        'rarity': 'legendary',
        'requirement': {'total_reports': 1000},
        'points_awarded': 500,
        'notification': '👑 HALL OF FAME! 1,000 reports! Your contribution to AMR surveillance is extraordinary.',
    },
    
    # ─── Voice Reporting ─────────────────────────────────
    
    'voice_master': {
        'name': 'Voice Master',
        'icon': '🎤',
        'description': 'Submitted 10 voice reports',
        'category': 'voice',
        'rarity': 'common',
        'requirement': {'voice_reports': 10},
        'points_awarded': 15,
        'notification': '🎤 Voice Master! You\'ve mastered voice reporting. Keep talking, keep reporting!',
    },
    
    'multilingual': {
        'name': 'Multilingual Reporter',
        'icon': '🌍',
        'description': 'Submitted reports in 3+ languages',
        'category': 'voice',
        'rarity': 'uncommon',
        'requirement': {'languages_used': 3},
        'points_awarded': 20,
        'notification': '🌍 Multilingual Reporter! Reporting in {languages} — your voice reaches more communities.',
    },
    
    'voice_power': {
        'name': 'Voice Power',
        'icon': '⚡',
        'description': 'Submitted 50 voice reports',
        'category': 'voice',
        'rarity': 'rare',
        'requirement': {'voice_reports': 50},
        'points_awarded': 30,
        'notification': '⚡ Voice Power! 50 voice reports. You\'re showing that speaking saves lives.',
    },
    
    # ─── Photo Reporting ──────────────────────────────────
    
    'snap_detective': {
        'name': 'Snap Detective',
        'icon': '📸',
        'description': 'Submitted 10 photo reports with Snap-Detect',
        'category': 'photo',
        'rarity': 'common',
        'requirement': {'photo_reports': 10},
        'points_awarded': 15,
        'notification': '📸 Snap Detective! Your photos help us track drug quality and authenticity.',
    },
    
    'photo_master': {
        'name': 'Photo Master',
        'icon': '📷',
        'description': 'Submitted 50 photo reports',
        'category': 'photo',
        'rarity': 'uncommon',
        'requirement': {'photo_reports': 50},
        'points_awarded': 25,
        'notification': '📷 Photo Master! Your photo reports are building a visual database of drug labels.',
    },
    
    'drug_detective': {
        'name': 'Drug Detective',
        'icon': '🔍',
        'description': 'Identified 5 counterfeit or expired drugs',
        'category': 'photo',
        'rarity': 'rare',
        'requirement': {'counterfeit_detected': 5},
        'points_awarded': 50,
        'notification': '🔍 Drug Detective! You\'ve identified 5 counterfeit/expired drugs. You\'re protecting your community!',
    },
    
    # ─── Surveillance ─────────────────────────────────────
    
    'resistance_spotter': {
        'name': 'Resistance Spotter',
        'icon': '🔬',
        'description': 'First to identify a new resistance pattern in your district',
        'category': 'surveillance',
        'rarity': 'rare',
        'requirement': {'first_resistance_detection': 1},
        'points_awarded': 50,
        'notification': '🔬 Resistance Spotter! You were the first to detect a new resistance pattern in {district}. This is critical for public health!',
    },
    
    'outbreak_alerter': {
        'name': 'Outbreak Alerter',
        'icon': '🚨',
        'description': 'Reported 5+ cases that triggered outbreak alerts',
        'category': 'surveillance',
        'rarity': 'rare',
        'requirement': {'outbreak_alerts_triggered': 5},
        'points_awarded': 40,
        'notification': '🚨 Outbreak Alerter! Your vigilance has triggered 5 outbreak alerts. You\'re saving lives.',
    },
    
    'rare_pathogen': {
        'name': 'Rare Catch',
        'icon': '🦠',
        'description': 'Reported a rare or emerging pathogen',
        'category': 'surveillance',
        'rarity': 'epic',
        'requirement': {'rare_pathogen_reported': 1},
        'points_awarded': 75,
        'notification': '🦠 Rare Catch! You reported {pathogen} — a rare/emerging pathogen. Excellent surveillance!',
    },
    
    # ─── Training ────────────────────────────────────────
    
    'trained': {
        'name': 'Trained',
        'icon': '🎓',
        'description': 'Completed first training module',
        'category': 'training',
        'rarity': 'common',
        'requirement': {'training_modules_completed': 1},
        'points_awarded': 10,
        'notification': '🎓 Trained! You\'ve completed your first UDARA AI training module. Knowledge is power!',
    },
    
    'expert': {
        'name': 'Expert',
        'icon': '🎓',
        'description': 'Completed all training modules',
        'category': 'training',
        'rarity': 'uncommon',
        'requirement': {'training_modules_completed': 8},
        'points_awarded': 50,
        'notification': '🎓 Expert! You\'ve completed all training modules. You\'re now a certified AMR surveillance expert!',
    },
    
    'quiz_master': {
        'name': 'Quiz Master',
        'icon': '🧠',
        'description': 'Scored 100% on 3 training quizzes',
        'category': 'training',
        'rarity': 'rare',
        'requirement': {'perfect_quizzes': 3},
        'points_awarded': 30,
        'notification': '🧠 Quiz Master! 3 perfect quiz scores. Your AMR knowledge is excellent!',
    },
    
    # ─── Social ──────────────────────────────────────────
    
    'team_player': {
        'name': 'Team Player',
        'icon': '🤝',
        'description': 'Participated in 3 group challenges',
        'category': 'social',
        'rarity': 'common',
        'requirement': {'group_challenges_completed': 3},
        'points_awarded': 15,
        'notification': '🤝 Team Player! You\'ve completed 3 group challenges. Together we\'re stronger.',
    },
    
    'rising_star': {
        'name': 'Rising Star',
        'icon': '🌟',
        'description': 'New CHW with 10+ reports in first month',
        'category': 'social',
        'rarity': 'uncommon',
        'requirement': {'new_chw_month1_reports': 10},
        'points_awarded': 20,
        'notification': '🌟 Rising Star! 10 reports in your first month! You\'re off to an amazing start.',
    },
    
    'recruiter': {
        'name': 'Recruiter',
        'icon': '📢',
        'description': 'Referred 5 CHWs to UDARA AI',
        'category': 'social',
        'rarity': 'uncommon',
        'requirement': {'referrals_completed': 5},
        'points_awarded': 25,
        'notification': '📢 Recruiter! 5 CHWs joined through your referral. You\'re growing the network!',
    },
    
    # ─── Guardian Series (Progressive Tiers) ──────────────
    
    'guardian_bronze': {
        'name': 'Bronze Guardian',
        'icon': '🛡️',
        'description': 'Reached 100 total points',
        'category': 'guardian',
        'rarity': 'common',
        'requirement': {'total_points': 100},
        'points_awarded': 0,
        'notification': '🛡️ Bronze Guardian! You\'ve earned 100 points. Welcome to the Guardian ranks!',
    },
    
    'guardian_silver': {
        'name': 'Silver Guardian',
        'icon': '🛡️',
        'description': 'Reached 500 total points',
        'category': 'guardian',
        'rarity': 'uncommon',
        'requirement': {'total_points': 500},
        'points_awarded': 0,
        'notification': '🛡️ Silver Guardian! 500 points earned. Your dedication is recognized.',
    },
    
    'guardian_gold': {
        'name': 'Gold Guardian',
        'icon': '🛡️',
        'description': 'Reached 1500 total points',
        'category': 'guardian',
        'rarity': 'rare',
        'requirement': {'total_points': 1500},
        'points_awarded': 0,
        'notification': '🛡️ Gold Guardian! 1500 points. You are a pillar of AMR surveillance.',
    },
    
    'guardian_platinum': {
        'name': 'Platinum Guardian',
        'icon': '💎',
        'description': 'Reached 3000 total points',
        'category': 'guardian',
        'rarity': 'epic',
        'requirement': {'total_points': 3000},
        'points_awarded': 0,
        'notification': '💎 Platinum Guardian! Elite status. Top 5% of all UDARA AI CHWs.',
    },
    
    'guardian_diamond': {
        'name': 'Diamond Guardian',
        'icon': '👑',
        'description': 'Reached 5000 total points',
        'category': 'guardian',
        'rarity': 'legendary',
        'requirement': {'total_points': 5000},
        'points_awarded': 0,
        'notification': '👑 DIAMOND GUARDIAN! The highest rank. You are an AMR surveillance legend.',
    },
}
```

### 5.3 Badge Rarity Tiers

| Rarity | Drop Rate | Visual | Examples |
|--------|-----------|--------|---------|
| Common | 90%+ earn | Gray border | First Report, Trained |
| Uncommon | 50-90% earn | Green border | 10 Reports, Voice Master |
| Rare | 10-50% earn | Blue border | 100 Reports, Resistance Spotter |
| Epic | 1-10% earn | Purple border | 500 Reports, Rare Pathogen |
| Legendary | < 1% earn | Gold animated | 1000 Reports, Diamond Guardian |

### 5.4 Badge Notification System

```python
class BadgeNotificationService:
    """Send badge unlock notifications via preferred channel."""
    
    TEMPLATES = {
        'whatsapp': '🎉 *NEW BADGE: {icon} {name}*\n\n{description}\n\n+{points} points!\n\nShare your achievement: /sharebadge {badge_id}',
        'telegram': '🎉 NEW BADGE: {icon} {name}\n\n{description}\n\n+{points} points!\n\n[Share Achievement](/sharebadge {badge_id})',
        'ussd': 'CON Badge: {icon} {name}!\n+{pts} pts.',
        'sms': 'UDARA AI: 🎉 New badge - {icon} {name}! +{pts} pts. Keep reporting!',
    }
    
    async def send_badge_notification(self, chw_id: str,
                                       badge_id: str,
                                       channel: str):
        """Send badge unlock notification."""
        badge = BADGE_DEFINITIONS[badge_id]
        chw = await self.db.get_chw(chw_id)
        
        template = self.TEMPLATES.get(channel, self.TEMPLATES['sms'])
        message = template.format(
            icon=badge['icon'],
            name=badge['name'],
            description=badge['description'],
            points=badge['points_awarded'],
            badge_id=badge_id,
        )
        
        await self.notification_service.send(
            channel=channel,
            recipient=chw['phone'],
            message=message,
        )
        
        # Store notification record
        await self.db.create_notification(
            chw_id=chw_id,
            type='badge_unlock',
            badge_id=badge_id,
            channel=channel,
        )
```

---

## 6. Leaderboards

### 6.1 Leaderboard Types

| Type | Scope | Timeframe | Categories |
|------|-------|-----------|------------|
| District | Your district | Weekly/Monthly/All-time | Points, Reports, Accuracy |
| Regional | Your region | Monthly/All-time | Points, Reports |
| National | Country-wide | Weekly/Monthly/All-time | Points, Reports, Accuracy |
| Team | Your team | Weekly | Team Points, Participation |

### 6.2 Leaderboard Mechanics

```
┌──────────────────────────────────────────────────────────────────┐
│                 LEADERBOARD DISPLAY (WhatsApp)                    │
│                                                                  │
│  ═══════════════════════════════════                             │
│  📊 DISTRICT LEADERBOARD — This Week                            │
│  Kibera District, Nairobi                                       │
│  ═══════════════════════════════════                             │
│                                                                  │
│  🥇 Mary Wanjiru        142 pts  📈 +12                          │
│  🥈 John Ochieng        128 pts  📈 +8                           │
│  🥉 Grace Akinyi        115 pts  📊 =                            │
│   4. Peter Mwangi       108 pts  📉 -3                           │
│   5. Fatuma Hassan      97 pts   📈 +15                          │
│  ───────────────────────────────                               │
│  📍 You: Rank #8       72 pts   📈 +5                           │
│  ───────────────────────────────                               │
│  6 pts to reach Rank #5! Report today!                         │
│                                                                  │
│  🏆 Your badges: 🏥📊🎤📸🔥                                     │
│  ⏰ Streak: 12 days | Tier: Silver Guardian                    │
│                                                                  │
│  Reply:                                                         │
│  1. My Stats    2. Full Board    3. National Rank               │
│  4. Team Board  5. This Month   6. Help                        │
│  ═══════════════════════════════════                             │
└──────────────────────────────────────────────────────────────────┘
```

### 6.3 Anti-Toxicity Measures

| Concern | Mitigation |
|---------|-----------|
| Demotivation from being at bottom | Show "personal best" not just rank; celebrate improvement |
| Cheating to climb leaderboard | Anti-gaming detection; supervisor verification |
| Unhealthy competition | Team challenges alongside individual; promote collaboration |
| Privacy concerns | Leaderboard only shows first name + initial; opt-out available |
| Geographic unfairness (easy vs hard districts) | Normalize by district reporting rate |

---

## 7. Rewards Tiers

### 7.1 Tier Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                    REWARDS TIER PROGRESSION                       │
│                                                                  │
│  Points                                                          │
│   5000 ┤                              ║ 💎 PLATINUM GUARDIAN       │
│        │                              ║ $10 airtime/month          │
│        │                              ║ Conference attendance      │
│   3000 ┤                              ║ Featured in newsletter      │
│        │                        ║ 🛡️ GOLD GUARDIAN              │
│        │                        ║ $5 airtime/month              │
│   1500 ┤────────────────────────╫ Branded UDARA shirt          │
│        │                  ║ 🛡️ SILVER GUARDIAN                  │
│        │                  ║ $2 airtime/month                    │
│    500 ┤──────────────────╫ Digital certificate                │
│        │            ║ 🛡️ BRONZE GUARDIAN                         │
│        │            ║ Recognition badge                        │
│    100 ┤───────────╫                                          │
│        │      ║                                                   │
│      0 ┼──────╫──────┬────────┬────────┬────────┬────────►       │
│        Bronze   Silver  Gold     Platinum  Diamond               │
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │ TIER DETAILS                                            │    │
│  │                                                         │    │
│  │ BRONZE (0-100 pts):                                    │    │
│  │ • Digital "Health Guardian" certificate                 │    │
│  │ • Recognition in district WhatsApp group               │    │
│  │ • Access to basic training modules                      │    │
│  │                                                         │    │
│  │ SILVER (100-500 pts):                                  │    │
│  │ • All Bronze benefits                                  │    │
│  │ • $2 USD phone airtime credit per month                 │    │
│  │ • Priority support channel                             │    │
│  │ • Access to advanced training modules                  │    │
│  │ • Monthly progress report                              │    │
│  │                                                         │    │
│  │ GOLD (500-1500 pts):                                   │    │
│  │ • All Silver benefits                                  │    │
│  │ • $5 USD phone airtime credit per month                 │    │
│  │ • Branded UDARA AI t-shirt or cap                       │    │
│  │ • Invited to quarterly webinars                        │    │
│  │ • Featured in district newsletter                     │    │
│  │                                                         │    │
│  │ PLATINUM (1500-3000+ pts):                             │    │
│  │ • All Gold benefits                                    │    │
│  │ • $10 USD phone airtime credit per month                │    │
│  │ • Invitation to annual AMR conference                   │    │
│  │ • Featured in national newsletter                      │    │
│  │ • Mentorship program access                            │    │
│  │                                                         │    │
│  │ DIAMOND (5000+ pts):                                   │    │
│  │ • All Platinum benefits                                │    │
│  │ • $15 USD airtime credit per month                     │    │
│  │ • All-expense-paid conference attendance                │    │
│  │ • "UDARA AI Champion" title                             │    │
│  │ • Letter of recognition from Ministry of Health         │    │
│  │ • Invited to advisory board                            │    │
│  └─────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### 7.2 Reward Catalog

| Reward | Points Cost | Real Cost (USD) | Redemption Rate |
|--------|-------------|-----------------|----------------|
| Phone airtime $2 | 100 pts | $2.00 | 45% |
| Phone airtime $5 | 250 pts | $5.00 | 22% |
| Phone airtime $10 | 450 pts | $10.00 | 12% |
| Data bundle 1GB | 150 pts | $1.50 | 8% |
| Data bundle 5GB | 350 pts | $4.00 | 3% |
| UDARA t-shirt | 300 pts | $8.00 | 5% |
| UDARA cap | 250 pts | $5.00 | 4% |
| Hand sanitizer pack | 200 pts | $3.00 | 6% |
| Medical kit (basic) | 500 pts | $12.00 | 2% |
| Conference attendance | 1000 pts | $200.00 | 0.5% |
| Certificate (framed) | 150 pts | $2.00 | 7% |

### 7.3 Reward Fulfillment Pipeline

```python
class RewardFulfillmentService:
    """
    Fulfill reward redemptions.
    
    Airtime rewards: Automated via mobile money API
    Physical rewards: Shipped via local logistics partner
    Digital rewards: Instant via email/SMS
    """
    
    FULFILLMENT_HANDLERS = {
        'airtime': '_fulfill_airtime',
        'data_bundle': '_fulfill_data_bundle',
        'tshirt': '_fulfill_physical',
        'cap': '_fulfill_physical',
        'certificate': '_fulfill_digital',
        'conference': '_fulfill_event',
    }
    
    async def redeem_reward(self, chw_id: str,
                             reward_id: str) -> dict:
        """Process a reward redemption request."""
        chw = await self.db.get_chw(chw_id)
        reward = await self.db.get_reward(reward_id)
        
        # Check if CHW has enough points
        if chw['total_points'] < reward['points_cost']:
            return {'status': 'error', 'message': 'Insufficient points'}
        
        # Deduct points
        await self.db.deduct_points(chw_id, reward['points_cost'], 'reward_redemption')
        
        # Fulfill
        handler_name = self.FULFILLMENT_HANDLERS.get(reward['type'])
        if handler_name:
            handler = getattr(self, handler_name)
            result = await handler(chw, reward)
        else:
            result = {'status': 'error', 'message': 'Unknown reward type'}
        
        # Log redemption
        await self.db.create_redemption(
            chw_id=chw_id,
            reward_id=reward_id,
            points_spent=reward['points_cost'],
            fulfillment_result=result,
        )
        
        # Notify CHW
        await self._send_redemption_notification(chw, reward, result)
        
        return result
    
    async def _fulfill_airtime(self, chw, reward) -> dict:
        """Fulfill airtime reward via mobile money API."""
        amount = reward['metadata']['amount_usd']
        phone = chw['phone']
        
        # Integrate with M-Pesa / Airtel Money / MTN Mobile Money
        result = await self.mobile_money_client.send_airtime(
            phone=phone,
            amount=amount,
            reference=f"UDARA-{chw['id']}-{reward['id']}",
        )
        
        if result['success']:
            return {
                'status': 'fulfilled',
                'method': 'mobile_money',
                'amount': amount,
                'transaction_id': result['transaction_id'],
            }
        else:
            # Queue for retry
            return {
                'status': 'pending',
                'reason': result['error'],
                'retry_after': '24h',
            }
```

### 7.4 Budget Management

```python
class BudgetManager:
    """
    Manage the gamification rewards budget.
    
    Ensures the program stays within budget while maximizing impact.
    """
    
    BUDGET_ALLOCATION = {
        'airtime_rewards': 0.50,     # 50% of budget
        'physical_rewards': 0.15,    # 15% of budget
        'event_rewards': 0.20,       # 20% of budget
        'operational': 0.10,         # 10% of budget
        'contingency': 0.05,         # 5% reserve
    }
    
    def calculate_monthly_budget(self, 
                                  total_annual_budget_usd: float,
                                  n_active_chws: int) -> dict:
        """
        Calculate monthly budget allocation.
        """
        monthly = total_annual_budget_usd / 12
        
        return {
            'annual_total': total_annual_budget_usd,
            'monthly_total': monthly,
            'per_chw_budget': monthly / n_active_chws,
            'airtime': monthly * self.BUDGET_ALLOCATION['airtime_rewards'],
            'physical': monthly * self.BUDGET_ALLOCATION['physical_rewards'],
            'events': monthly * self.BUDGET_ALLOCATION['event_rewards'],
            'operational': monthly * self.BUDGET_ALLOCATION['operational'],
            'contingency': monthly * self.BUDGET_ALLOCATION['contingency'],
        }
    
    def forecast_spend(self, current_month_spending: dict,
                        n_active_chws: int,
                        month_progress: float) -> dict:
        """
        Forecast end-of-month spending and recommend adjustments.
        """
        projected_total = {}
        for category, spent in current_month_spending.items():
            projected = spent / month_progress  # Extrapolate to full month
            projected_total[category] = projected
        
        total_projected = sum(projected_total.values())
        
        # Recommend adjustments if over-budget
        recommendations = []
        for category, projected in projected_total.items():
            budget = self.BUDGET_ALLOCATION.get(category, 0) * sum(current_month_spending.values()) / month_progress
            if projected > budget * 1.1:  # 10% over
                recommendations.append({
                    'category': category,
                    'status': 'over_budget',
                    'projected': projected,
                    'budget': budget,
                    'recommendation': f'Reduce {category} spending or reallocate from contingency',
                })
        
        return {
            'projected_spend': projected_total,
            'total_projected': total_projected,
            'recommendations': recommendations,
        }
```

---

## 8. Technical Implementation

### 8.1 Data Model

```sql
-- ============================================================
-- GAMIFICATION DATABASE SCHEMA
-- ============================================================

-- CHW gamification profile
CREATE TABLE gamification_profiles (
    chw_id             TEXT PRIMARY KEY REFERENCES chw(id),
    total_points       INTEGER NOT NULL DEFAULT 0,
    current_streak     INTEGER NOT NULL DEFAULT 0,
    longest_streak     INTEGER NOT NULL DEFAULT 0,
    last_active_date  DATE,
    current_tier      TEXT NOT NULL DEFAULT 'bronze',
    tier_reached_date  DATE,
    reports_count     INTEGER NOT NULL DEFAULT 0,
    voice_reports     INTEGER NOT NULL DEFAULT 0,
    photo_reports     INTEGER NOT NULL DEFAULT 0,
    languages_used    JSON DEFAULT '[]',
    opt_out_leaderboard BOOLEAN NOT NULL DEFAULT FALSE,
    created_at        TIMESTAMP DEFAULT NOW(),
    updated_at        TIMESTAMP DEFAULT NOW()
);

-- Points transaction log
CREATE TABLE points_log (
    id                 SERIAL PRIMARY KEY,
    chw_id             TEXT NOT NULL REFERENCES gamification_profiles(chw_id),
    points_change      INTEGER NOT NULL,  -- positive or negative
    running_total      INTEGER NOT NULL,
    action_type        TEXT NOT NULL,  -- 'report', 'badge', 'streak', etc.
    action_detail      JSON,           -- additional context
    multiplier         FLOAT DEFAULT 1.0,
    metadata           JSON,
    created_at         TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_points_chw ON points_log(chw_id);
CREATE INDEX idx_points_date ON points_log(created_at);

-- Badges earned
CREATE TABLE badges_earned (
    id                 SERIAL PRIMARY KEY,
    chw_id             TEXT NOT NULL REFERENCES gamification_profiles(chw_id),
    badge_id           TEXT NOT NULL,
    earned_at          TIMESTAMP DEFAULT NOW(),
    notification_sent  BOOLEAN DEFAULT FALSE,
    shared             BOOLEAN DEFAULT FALSE,
    UNIQUE(chw_id, badge_id)
);

-- Leaderboard snapshots (for historical tracking)
CREATE TABLE leaderboard_snapshots (
    id                 SERIAL PRIMARY KEY,
    snapshot_date      DATE NOT NULL,
    scope              TEXT NOT NULL,  -- 'district', 'regional', 'national'
    scope_id           TEXT NOT NULL,  -- district_id, region_id, etc.
    timeframe          TEXT NOT NULL,  -- 'weekly', 'monthly', 'all_time'
    ranking_data      JSON NOT NULL,  -- Array of {chw_id, rank, points}
    created_at         TIMESTAMP DEFAULT NOW(),
    UNIQUE(snapshot_date, scope, scope_id, timeframe)
);

-- Streak history
CREATE TABLE streaks (
    id                 SERIAL PRIMARY KEY,
    chw_id             TEXT NOT NULL REFERENCES gamification_profiles(chw_id),
    streak_length      INTEGER NOT NULL,
    started_at         DATE NOT NULL,
    ended_at           DATE NOT NULL,
    break_reason       TEXT  -- 'gap', 'inactive', etc.
);

-- Rewards catalog
CREATE TABLE rewards_catalog (
    id                 SERIAL PRIMARY KEY,
    reward_id          TEXT UNIQUE NOT NULL,
    name               TEXT NOT NULL,
    description        TEXT,
    points_cost        INTEGER NOT NULL,
    reward_type        TEXT NOT NULL,  -- 'airtime', 'physical', 'digital', 'event'
    real_cost_usd      FLOAT,
    metadata           JSON,
    available          BOOLEAN DEFAULT TRUE,
    max_per_month      INTEGER DEFAULT 0,  -- 0 = unlimited
    max_redemptions    INTEGER DEFAULT 0,
    created_at         TIMESTAMP DEFAULT NOW()
);

-- Rewards redeemed
CREATE TABLE rewards_claimed (
    id                 SERIAL PRIMARY KEY,
    chw_id             TEXT NOT NULL,
    reward_id          TEXT NOT NULL REFERENCES rewards_catalog(reward_id),
    points_spent       INTEGER NOT NULL,
    fulfillment_status TEXT DEFAULT 'pending',  -- pending, fulfilled, failed
    fulfillment_data   JSON,
    claimed_at         TIMESTAMP DEFAULT NOW(),
    fulfilled_at       TIMESTAMP
);

-- Multiplier events
CREATE TABLE multiplier_events (
    id                 SERIAL PRIMARY KEY,
    event_id           TEXT UNIQUE NOT NULL,
    name               TEXT NOT NULL,
    multiplier         FLOAT NOT NULL DEFAULT 1.0,
    start_date         DATE,
    end_date           DATE,
    scope              TEXT DEFAULT 'all',
    scope_id           TEXT DEFAULT NULL,
    is_active          BOOLEAN DEFAULT FALSE
);

-- Anti-gaming: cooldown tracking
CREATE TABLE reporting_cooldowns (
    chw_id             TEXT NOT NULL,
    cooldown_date      DATE NOT NULL,
    reports_count      INTEGER NOT NULL DEFAULT 0,
    last_report_time   TIMESTAMP,
    PRIMARY KEY (chw_id, cooldown_date)
);
```

### 8.2 Gamification Service

```python
class GamificationService:
    """
    Core gamification engine.
    
    Processes all actions that earn points, checks badges,
    updates leaderboards, and manages streaks.
    """
    
    POINTS_MAP = {
        'report_text': 10,
        'report_voice': 8,
        'report_photo': 15,
        'report_snap_detect': 12,
        'report_complete': 5,
        'report_verified': 10,
        'daily_first_report': 3,
        'first_in_district_today': 5,
        'resistance_detected': 50,
        'refer_chw': 25,
        'training_module': 30,
        'quiz_pass': 15,
        'quiz_perfect': 15,
        'refresher_attend': 20,
        'holiday_report': 10,
        'rare_pathogen': 15,
        'lab_confirmed': 8,
        'hard_to_reach': 5,
        'gps_report': 3,
        # Penalties
        'duplicate_report': -5,
        'report_rejected': -2,
        'cooldown_violation': -10,
    }
    
    MAX_DAILY_REPORTS = 20
    COOLDOWN_MINUTES = 3  # Min 3 minutes between reports
    
    def __init__(self, db, notification_service, leaderboard_engine):
        self.db = db
        self.notifications = notification_service
        self.leaderboards = leaderboard_engine
        self.badge_engine = BadgeEngine(db)
        self.streak_engine = StreakEngine()
        self.multiplier_engine = MultiplierEngine()
    
    async def award_points(self, 
                            chw_id: str, 
                            action: str, 
                            metadata: dict = None) -> dict:
        """
        Award points for an action.
        
        This is the MAIN ENTRY POINT called by other services when
        a CHW performs a reportable action.
        """
        import datetime
        
        base_points = self.POINTS_MAP.get(action, 0)
        if base_points == 0:
            return {'status': 'no_action', 'points': 0}
        
        # Check anti-gaming: cooldown
        if action.startswith('report'):
            cooldown_check = await self._check_cooldown(chw_id)
            if not cooldown_check['allowed']:
                return {
                    'status': 'cooldown',
                    'message': cooldown_check['message'],
                    'points': 0,
                }
        
        # Calculate streak bonus
        profile = await self.db.get_gamification_profile(chw_id)
        streak_bonus = self.streak_engine.calculate_streak_bonus(
            current_streak_days=profile['current_streak'],
            reports_today=1,
        )
        
        # Calculate multipliers
        today = datetime.date.today().isoformat()
        multipliers = self.multiplier_engine.get_active_multipliers(today)
        total_multiplier = 1.0
        for m in multipliers:
            total_multiplier *= m['multiplier']
        
        # Calculate total
        total = base_points
        total += streak_bonus['total_bonus']
        total = int(total * total_multiplier)
        
        # Update database
        await self.db.add_points(
            chw_id=chw_id,
            points=total,
            action=action,
            metadata=metadata,
            multiplier=total_multiplier,
        )
        
        # Update streak
        await self._update_streak(chw_id, profile)
        
        # Check badge unlocks
        new_badges = await self.badge_engine.check_and_award(chw_id)
        
        # Update leaderboard
        rank_change = await self.leaderboards.update_ranking(chw_id)
        
        # Send notifications for new badges
        for badge_id in new_badges:
            await self.notifications.send_badge_notification(
                chw_id, badge_id, profile.get('preferred_channel', 'whatsapp')
            )
        
        return {
            'status': 'success',
            'points': total,
            'base_points': base_points,
            'streak_bonus': streak_bonus['total_bonus'],
            'multiplier': total_multiplier,
            'active_multipliers': multipliers,
            'new_badges': new_badges,
            'rank_change': rank_change,
            'current_streak': profile['current_streak'] + 1,
            'total_points': profile['total_points'] + total,
        }
    
    async def _check_cooldown(self, chw_id: str) -> dict:
        """Check if CHW is in cooldown."""
        import datetime
        
        today = datetime.date.today().isoformat()
        cooldown = await self.db.get_cooldown(chw_id, today)
        
        if cooldown and cooldown['reports_count'] >= self.MAX_DAILY_REPORTS:
            return {
                'allowed': False,
                'message': f'You\'ve reached the daily limit ({self.MAX_DAILY_REPORTS} reports). '
                          f'Your reports are still recorded but bonus points are paused for today.',
                'is_penalty': False,
            }
        
        # Check inter-report cooldown
        last_report = await self.db.get_last_report_time(chw_id)
        if last_report:
            elapsed = (datetime.datetime.utcnow() - last_report).total_seconds() / 60
            if elapsed < self.COOLDOWN_MINUTES:
                return {
                    'allowed': False,
                    'message': f'Please wait {self.COOLDOWN_MINUTES - int(elapsed)} minutes '
                              f'between reports.',
                    'is_penalty': False,
                }
        
        return {'allowed': True}
    
    async def _update_streak(self, chw_id: str, profile: dict):
        """Update the CHW's reporting streak."""
        import datetime
        
        today = datetime.date.today()
        last_active = datetime.datetime.strptime(
            profile['last_active_date'], '%Y-%m-%d'
        ).date() if profile['last_active_date'] else None
        
        if last_active == today:
            return  # Already active today
        
        # Check if streak continues
        if last_active and (today - last_active).days == 1:
            # Streak continues
            new_streak = profile['current_streak'] + 1
        elif last_active == today - datetime.timedelta(days=0):
            pass  # Same day, no update needed
        else:
            # Streak broken — save to history and restart
            if profile['current_streak'] > 1:
                await self.db.save_streak_history(
                    chw_id=chw_id,
                    length=profile['current_streak'],
                    ended_at=last_active,
                    break_reason='gap',
                )
            new_streak = 1
        
        await self.db.update_streak(chw_id, new_streak, today)
        
        if new_streak > profile['longest_streak']:
            await self.db.update_longest_streak(chw_id, new_streak)
```

### 8.3 Badge Engine

```python
class BadgeEngine:
    """Check and award badges based on CHW activity."""
    
    def __init__(self, db):
        self.db = db
    
    async def check_and_award(self, chw_id: str) -> List[str]:
        """
        Check all badge conditions and award any new badges.
        
        Returns:
            List of newly awarded badge IDs
        """
        profile = await self.db.get_gamification_profile(chw_id)
        stats = await self.db.get_chw_stats(chw_id)
        
        new_badges = []
        
        for badge_id, badge_def in BADGE_DEFINITIONS.items():
            # Skip if already earned
            if await self.db.has_badge(chw_id, badge_id):
                continue
            
            # Check requirements
            if self._meets_requirements(badge_def['requirement'], profile, stats):
                await self.db.award_badge(chw_id, badge_id)
                new_badges.append(badge_id)
        
        return new_badges
    
    def _meets_requirements(self, requirement: dict,
                             profile: dict,
                             stats: dict) -> bool:
        """Check if a CHW meets badge requirements."""
        for key, value in requirement.items():
            if key == 'total_reports':
                if profile['reports_count'] < value:
                    return False
            elif key == 'voice_reports':
                if profile['voice_reports'] < value:
                    return False
            elif key == 'photo_reports':
                if profile['photo_reports'] < value:
                    return False
            elif key == 'total_points':
                if profile['total_points'] < value:
                    return False
            elif key == 'languages_used':
                if len(profile.get('languages_used', [])) < value:
                    return False
            elif key == 'first_resistance_detection':
                if stats.get('resistance_first_detections', 0) < value:
                    return False
            elif key == 'training_modules_completed':
                if stats.get('training_completed', 0) < value:
                    return False
            elif key == 'perfect_quizzes':
                if stats.get('perfect_quizzes', 0) < value:
                    return False
            elif key == 'group_challenges_completed':
                if stats.get('challenges_completed', 0) < value:
                    return False
            elif key == 'referrals_completed':
                if stats.get('successful_referrals', 0) < value:
                    return False
            elif key == 'counterfeit_detected':
                if stats.get('counterfeit_flagged', 0) < value:
                    return False
            elif key == 'outbreak_alerts_triggered':
                if stats.get('alerts_triggered', 0) < value:
                    return False
            elif key == 'rare_pathogen_reported':
                if stats.get('rare_pathogens', 0) < value:
                    return False
        
        return True
```

### 8.4 Leaderboard Engine

```python
class LeaderboardEngine:
    """
    Compute and cache leaderboard rankings.
    
    Updated every 15 minutes via background task.
    """
    
    async def update_ranking(self, chw_id: str) -> dict:
        """Update a CHW's position on all relevant leaderboards."""
        profile = await self.db.get_gamification_profile(chw_id)
        district_id = profile.get('district_id')
        
        rank_changes = {}
        
        for scope in ['district']:
            scope_id = district_id if scope == 'district' else None
            for timeframe in ['weekly', 'monthly']:
                old_rank = await self.db.get_leaderboard_rank(
                    chw_id, scope, scope_id, timeframe
                )
                
                # Recompute ranking
                await self._compute_leaderboard(scope, scope_id, timeframe)
                
                new_rank = await self.db.get_leaderboard_rank(
                    chw_id, scope, scope_id, timeframe
                )
                
                if old_rank and new_rank:
                    rank_changes[f"{scope}_{timeframe}"] = {
                        'old_rank': old_rank,
                        'new_rank': new_rank,
                        'change': old_rank - new_rank,  # Positive = improved
                    }
        
        return rank_changes
    
    async def _compute_leaderboard(self, scope: str, 
                                    scope_id: str,
                                    timeframe: str):
        """Compute leaderboard from scratch."""
        # Query top reporters based on timeframe
        if timeframe == 'weekly':
            date_filter = "created_at >= NOW() - INTERVAL '7 days'"
        elif timeframe == 'monthly':
            date_filter = "created_at >= NOW() - INTERVAL '30 days'"
        else:
            date_filter = "1=1"
        
        query = f"""
        SELECT 
            chw_id,
            SUM(points_change) as total_points,
            COUNT(*) as report_count
        FROM points_log
        WHERE {date_filter}
        GROUP BY chw_id
        ORDER BY total_points DESC
        LIMIT 100
        """
        
        rankings = await self.db.execute(query)
        
        # Store snapshot
        await self.db.save_leaderboard_snapshot(
            scope=scope,
            scope_id=scope_id,
            timeframe=timeframe,
            rankings=rankings,
        )
    
    async def get_leaderboard(self, scope: str,
                               scope_id: str,
                               timeframe: str,
                               viewer_chw_id: str = None,
                               limit: int = 10) -> dict:
        """Get leaderboard for display."""
        snapshot = await self.db.get_latest_snapshot(
            scope, scope_id, timeframe
        )
        
        if not snapshot:
            await self._compute_leaderboard(scope, scope_id, timeframe)
            snapshot = await self.db.get_latest_snapshot(
                scope, scope_id, timeframe
            )
        
        rankings = snapshot['ranking_data'][:limit]
        
        # Add viewer's own rank if not in top N
        viewer_rank = None
        if viewer_chw_id:
            for r in rankings:
                if r['chw_id'] == viewer_chw_id:
                    viewer_rank = r['rank']
                    break
        
        return {
            'scope': scope,
            'scope_id': scope_id,
            'timeframe': timeframe,
            'rankings': rankings,
            'viewer_rank': viewer_rank,
            'updated_at': snapshot['created_at'],
        }
```

---

## 9. Integration Points

### 9.1 USSD Integration

```
CHW dials: *123# → selects option 2

CON UDARA GAMIFICATION
─────────────────────
1. My Points
2. My Badges
3. Leaderboard
4. Redeem Reward
5. My Streak
─────────────────────

CHW enters: 1

CON Your Points
───────────────
Total: 342 pts
Today: +12 pts
Tier: Silver Guard
Next tier: Gold (1158 pts)
Streak: 12 days
───────────────
0.Back
```

### 9.2 WhatsApp Integration

```python
class WhatsAppGamificationBot:
    """WhatsApp bot commands for gamification."""
    
    COMMANDS = {
        '/points': 'show_points',
        '/badges': 'show_badges',
        '/leaderboard': 'show_leaderboard',
        '/streak': 'show_streak',
        '/redeem': 'show_reward_catalog',
        '/rank': 'show_rank',
        '/stats': 'show_full_stats',
        '/share': 'share_achievement',
    }
    
    async def handle_command(self, phone: str, 
                               command: str,
                               args: str = '') -> str:
        """Handle gamification commands from WhatsApp."""
        chw_id = await self.db.get_chw_by_phone(phone)
        
        handler_name = self.COMMANDS.get(command)
        if not handler_name:
            return "Unknown command. Type /help for available commands."
        
        handler = getattr(self, handler_name)
        return await handler(chw_id, args)
    
    async def show_points(self, chw_id: str, args: str) -> str:
        """Show CHW's points and tier."""
        profile = await self.db.get_gamification_profile(chw_id)
        
        tier_emoji = {'bronze': '🥉', 'silver': '🥈', 'gold': '🥇', 
                      'platinum': '💎', 'diamond': '👑'}
        
        next_tier_points = {
            'bronze': 100, 'silver': 500, 
            'gold': 1500, 'platinum': 3000, 'diamond': 5000,
        }
        
        current_tier = profile['current_tier']
        next_tier = {
            'bronze': 'silver', 'silver': 'gold',
            'gold': 'platinum', 'platinum': 'diamond',
        }.get(current_tier)
        
        pts_to_next = (
            next_tier_points[next_tier] - profile['total_points']
            if next_tier else 0
        )
        
        return (
            f"📊 *Your Points*\n"
            f"━━━━━━━━━━━━━━━━━━\n"
            f"Total: *{profile['total_points']}* pts\n"
            f"Streak: 🔥 *{profile['current_streak']}* days\n"
            f"Tier: {tier_emoji.get(current_tier, '')} *{current_tier.title()} Guardian*\n"
            f"{'Next: ' + next_tier.title() + f' ({pts_to_next} pts needed)' if next_tier else '🏆 MAX TIER!'}\n"
            f"━━━━━━━━━━━━━━━━━━\n"
            f"Keep reporting to earn more! 💪"
        )
```

### 9.3 Telegram Integration

```python
class TelegramGamificationBot:
    """Telegram inline keyboard for gamification."""
    
    async def send_stats_keyboard(self, chat_id: int, chw_id: str):
        """Send interactive stats with inline keyboard."""
        profile = await self.db.get_gamification_profile(chw_id)
        
        keyboard = InlineKeyboardMarkup([
            [InlineKeyboardButton("📊 My Points", callback_data="gam_points"),
             InlineKeyboardButton("🏆 Badges", callback_data="gam_badges")],
            [InlineKeyboardButton("📈 Leaderboard", callback_data="gam_leaderboard"),
             InlineKeyboardButton("🔥 Streak", callback_data="gam_streak")],
            [InlineKeyboardButton("🎁 Rewards", callback_data="gam_rewards"),
             InlineKeyboardButton("📊 Full Stats", callback_data="gam_full")],
        ])
        
        await self.bot.send_message(
            chat_id=chat_id,
            text=(
                f"🎮 *UDARA AI Gamification*\n\n"
                f"Points: *{profile['total_points']}*\n"
                f"Streak: 🔥 {profile['current_streak']} days\n"
                f"Tier: {profile['current_tier'].title()}\n"
            ),
            reply_markup=keyboard,
            parse_mode='Markdown',
        )
```

### 9.4 Web Dashboard

```javascript
// Web dashboard API endpoints

const GAMIFICATION_API = {
  // GET /api/v1/gamification/profile/:chw_id
  getProfile: (chwId) => `/api/v1/gamification/profile/${chwId}`,
  
  // GET /api/v1/gamification/points/:chw_id
  getPoints: (chwId) => `/api/v1/gamification/points/${chwId}`,
  
  // GET /api/v1/gamification/badges/:chw_id
  getBadges: (chwId) => `/api/v1/gamification/badges/${chwId}`,
  
  // GET /api/v1/gamification/leaderboard?scope=district&scope_id=KIBERA&timeframe=weekly
  getLeaderboard: (params) => `/api/v1/gamification/leaderboard?${new URLSearchParams(params)}`,
  
  // GET /api/v1/gamification/streak/:chw_id
  getStreak: (chwId) => `/api/v1/gamification/streak/${chwId}`,
  
  // GET /api/v1/gamification/rewards/catalog
  getRewardsCatalog: () => '/api/v1/gamification/rewards/catalog',
  
  // POST /api/v1/gamification/rewards/redeem
  redeemReward: (chwId, rewardId) => ({
    url: '/api/v1/gamification/rewards/redeem',
    method: 'POST',
    body: { chw_id: chwId, reward_id: rewardId },
  }),
  
  // GET /api/v1/gamification/analytics/program-manager
  getProgramAnalytics: () => '/api/v1/gamification/analytics/program-manager',
};
```

---

## 10. Anti-Gaming Measures

### 10.1 Duplicate Detection

```python
class DuplicateDetector:
    """Detect duplicate or near-duplicate reports."""
    
    SIMILARITY_THRESHOLD = 0.85  # 85% similarity = likely duplicate
    
    async def check_duplicate(self, report: dict) -> dict:
        """Check if this report is a duplicate of a recent one."""
        chw_id = report['chw_id']
        
        # Get recent reports from this CHW (last 24 hours)
        recent = await self.db.get_recent_reports(
            chw_id=chw_id, hours=24
        )
        
        for existing in recent:
            similarity = self._compute_similarity(report, existing)
            
            if similarity > self.SIMILARITY_THRESHOLD:
                return {
                    'is_duplicate': True,
                    'similar_report_id': existing['id'],
                    'similarity': similarity,
                    'action': 'reject_and_penalize',
                }
        
        # Cross-CHW duplicate check (same patient, different CHW)
        if report.get('patient_id'):
            cross_check = await self.db.get_reports_by_patient(
                patient_id=report['patient_id'],
                hours=72,
            )
            for existing in cross_check:
                if existing['chw_id'] != chw_id:
                    similarity = self._compute_similarity(report, existing)
                    if similarity > 0.90:
                        return {
                            'is_duplicate': True,
                            'similar_report_id': existing['id'],
                            'similarity': similarity,
                            'action': 'flag_for_review',  # Don't penalize, may be legitimate
                        }
        
        return {'is_duplicate': False}
    
    def _compute_similarity(self, report1: dict, report2: dict) -> float:
        """Compute similarity between two reports."""
        from difflib import SequenceMatcher
        
        score = 0.0
        n_fields = 0
        
        for field in ['drug_id', 'pathogen_id', 'patient_age', 'patient_sex']:
            v1 = str(report1.get(field, ''))
            v2 = str(report2.get(field, ''))
            if v1 and v2:
                score += SequenceMatcher(None, v1, v2).ratio()
                n_fields += 1
        
        # Time proximity
        time_diff = abs(
            (report1.get('report_date', datetime.now()) - 
             report2.get('report_date', datetime.now())).total_seconds()
        )
        if time_diff < 3600:  # Within 1 hour
            score += 1.0
            n_fields += 1
        
        return score / n_fields if n_fields > 0 else 0.0
```

### 10.2 Quality Scoring

```python
class ReportQualityScorer:
    """
    Score report quality to determine bonus eligibility.
    Reports with all "unknown" fields get 0 bonus.
    """
    
    FIELD_WEIGHTS = {
        'drug_id': 0.25,
        'pathogen_id': 0.25,
        'patient_age': 0.10,
        'patient_sex': 0.05,
        'resistance_status': 0.20,
        'treatment_given': 0.10,
        'outcome': 0.05,
    }
    
    def score(self, report: dict) -> float:
        """Compute quality score (0-1)."""
        total_weight = 0
        total_score = 0
        
        for field, weight in self.FIELD_WEIGHTS.items():
            value = report.get(field)
            if value and value not in ('unknown', 'N/A', '', None):
                total_score += weight
            total_weight += weight
        
        return total_score / total_weight if total_weight > 0 else 0.0
    
    def get_bonus_multiplier(self, quality_score: float) -> float:
        """Points multiplier based on quality."""
        if quality_score >= 0.9:
            return 1.0   # Full points
        elif quality_score >= 0.7:
            return 0.8   # 80% of points
        elif quality_score >= 0.5:
            return 0.5   # 50% of points
        else:
            return 0.0   # No bonus points (but report still recorded)
```

### 10.3 Cooldown Enforcement

See section 8.2 — MAX_DAILY_REPORTS = 20 and COOLDOWN_MINUTES = 3.

### 10.4 Supervisor Verification

```python
class SupervisorVerification:
    """
    Random and targeted supervisor verification of reports.
    
    - 10% of reports are randomly flagged for supervisor review
    - Reports from CHWs with < 0.7 reliability score get 50% review
    - Reports that trigger outbreak alerts get 100% review
    """
    
    RANDOM_REVIEW_RATE = 0.10  # 10%
    
    async def should_review(self, report: dict, chw_reliability: float) -> bool:
        """Determine if a report should be reviewed by a supervisor."""
        # Outbreak alerts always reviewed
        if report.get('triggered_alert'):
            return True
        
        # Low reliability CHWs: 50% review rate
        if chw_reliability < 0.7:
            return random.random() < 0.50
        
        # Random review
        return random.random() < self.RANDOM_REVIEW_RATE
    
    async def submit_review(self, report_id: str,
                             reviewer_id: str,
                             is_accurate: bool,
                             notes: str = None):
        """Submit supervisor review and update CHW reliability."""
        await self.db.save_review(
            report_id=report_id,
            reviewer_id=reviewer_id,
            is_accurate=is_accurate,
            notes=notes,
        )
        
        # Update CHW reliability score
        chw_id = await self.db.get_report_chw(report_id)
        await self._update_reliability(chw_id, is_accurate)
    
    async def _update_reliability(self, chw_id: str, is_accurate: bool):
        """Exponential moving average of review outcomes."""
        reviews = await self.db.get_recent_reviews(chw_id, n=20)
        
        if len(reviews) < 5:
            return  # Not enough data
        
        accuracy_rate = sum(1 for r in reviews if r['is_accurate']) / len(reviews)
        await self.db.update_chw_reliability(chw_id, accuracy_rate)
```

### 10.5 Anomaly Detection

```python
class AnomalyDetector:
    """Detect suspicious gamification patterns."""
    
    SUSPICIOUS_PATTERNS = {
        'burst_reporting': {
            'check': 'More than 10 reports in 1 hour',
            'threshold': 10,
            'window_hours': 1,
            'action': 'FLAG',
        },
        'weekend_only': {
            'check': '95%+ reports on weekends (likely fake weekday data)',
            'threshold': 0.95,
            'action': 'FLAG',
        },
        'all_same_values': {
            'check': 'All reports have identical values',
            'threshold': 0.95,
            'action': 'FLAG',
        },
        'rapid_tier_jump': {
            'check': 'Gained 500+ points in 1 hour',
            'threshold': 500,
            'window_hours': 1,
            'action': 'FLAG',
        },
    }
    
    async def check(self, chw_id: str) -> List[dict]:
        """Run all anomaly checks for a CHW."""
        alerts = []
        
        # Burst reporting check
        recent = await self.db.get_recent_point_changes(
            chw_id, hours=1
        )
        if len(recent) > 10:
            alerts.append({
                'type': 'burst_reporting',
                'severity': 'medium',
                'message': f'{len(recent)} actions in the last hour',
            })
        
        # All same values check
        recent_reports = await self.db.get_recent_reports(chw_id, n=10)
        if recent_reports:
            drugs = [r['drug_id'] for r in recent_reports]
            if len(set(drugs)) == 1 and len(drugs) > 5:
                alerts.append({
                    'type': 'all_same_values',
                    'severity': 'medium',
                    'message': 'All recent reports have same drug',
                })
        
        return alerts
```

---

## 11. Analytics Dashboard

### 11.1 Program Manager Views

```python
# API endpoints for program manager analytics

DASHBOARD_QUERIES = {
    'engagement_rate': """
        SELECT 
            COUNT(DISTINCT chw_id) FILTER (WHERE last_active_date >= CURRENT_DATE - 7) 
            as active_chws,
            COUNT(DISTINCT chw_id) as total_chws,
            ROUND(COUNT(DISTINCT chw_id) FILTER (WHERE last_active_date >= CURRENT_DATE - 7)::numeric 
                  / COUNT(DISTINCT chw_id)::numeric * 100, 1) as engagement_pct
        FROM gamification_profiles;
    """,
    
    'avg_reports_per_chw': """
        SELECT 
            AVG(reports_count) as avg_total,
            PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY reports_count) as median_total,
            AVG(reports_count) FILTER (WHERE last_active_date >= CURRENT_DATE - 30) as avg_monthly
        FROM gamification_profiles;
    """,
    
    'points_distribution': """
        SELECT 
            current_tier,
            COUNT(*) as chw_count,
            AVG(total_points) as avg_points,
            MIN(total_points) as min_points,
            MAX(total_points) as max_points
        FROM gamification_profiles
        GROUP BY current_tier
        ORDER BY AVG(total_points);
    """,
    
    'top_performers': """
        SELECT 
            p.chw_id,
            c.name,
            c.district_id,
            p.total_points,
            p.current_streak,
            p.reports_count,
            p.current_tier
        FROM gamification_profiles p
        JOIN chw c ON p.chw_id = c.id
        WHERE p.last_active_date >= CURRENT_DATE - 7
        ORDER BY p.total_points DESC
        LIMIT 20;
    """,
    
    'badge_ownership': """
        SELECT 
            badge_id,
            COUNT(*) as earned_count,
            ROUND(COUNT(*)::numeric / (SELECT COUNT(*) FROM gamification_profiles)::numeric * 100, 1) as pct_earned
        FROM badges_earned
        GROUP BY badge_id
        ORDER BY earned_count DESC;
    """,
    
    'reward_redemption_rate': """
        SELECT 
            r.reward_id,
            rc.name,
            COUNT(*) as total_redemptions,
            SUM(rc.real_cost_usd) as total_spent,
            AVG(rc.points_cost) as avg_points_spent
        FROM rewards_claimed rc
        JOIN rewards_catalog r ON rc.reward_id = r.reward_id
        WHERE rc.fulfillment_status = 'fulfilled'
        GROUP BY r.reward_id, rc.name
        ORDER BY total_redemptions DESC;
    """,
}
```

---

## 12. Configuration Reference

| Variable | Default | Description |
|----------|---------|-------------|
| `UDARA_GAM_POINTS_REPORT_TEXT` | `10` | Points for text report |
| `UDARA_GAM_POINTS_REPORT_VOICE` | `8` | Points for voice report |
| `UDARA_GAM_POINTS_REPORT_PHOTO` | `15` | Points for photo report |
| `UDARA_GAM_MAX_DAILY_REPORTS` | `20` | Daily report cap |
| `UDARA_GAM_COOLDOWN_MINUTES` | `3` | Min time between reports |
| `UDARA_GAM_STREAK_DAILY_BONUS` | `5` | Daily streak base bonus |
| `UDARA_GAM_LEADERBOARD_UPDATE_MIN` | `15` | Leaderboard refresh interval |
| `UDARA_GAM_BADGE_CHECK_CRON` | `*/5 * * * *` | Badge check frequency (every 5 min) |
| `UDARA_GAM_POINT_DECAY_RATE` | `0.02` | Monthly point decay rate |
| `UDARA_GAM_POINT_DECAY_START` | `6` | Months before decay starts |
| `UDARA_GAM_SUPERVISOR_REVIEW_RATE` | `0.10` | Random review rate |

---

## 13. Testing Strategy

```python
import pytest

class TestGamificationService:
    
    def test_award_basic_points(self):
        """Test basic point awarding."""
        service = GamificationService(mock_db, mock_notif, mock_lb)
        result = await service.award_points('chw_001', 'report_text')
        assert result['status'] == 'success'
        assert result['points'] == 10
    
    def test_streak_bonus(self):
        """Test streak bonus calculation."""
        engine = StreakEngine()
        # Day 5 should get 8 bonus points
        result = engine.calculate_streak_bonus(current_streak_days=5, reports_today=1)
        assert result['daily_bonus'] == 8
    
    def test_max_daily_reports(self):
        """Test daily report limit enforcement."""
        service = GamificationService(mock_db, mock_notif, mock_lb)
        # Simulate 21 reports in one day
        for i in range(21):
            result = await service.award_points('chw_001', 'report_text')
        # Last report should be cooldown limited
        assert result['status'] == 'cooldown'
    
    def test_badge_unlock(self):
        """Test badge unlocking logic."""
        engine = BadgeEngine(mock_db)
        badges = await engine.check_and_award('chw_001')
        assert 'first_report' in badges
    
    def test_duplicate_detection(self):
        """Test duplicate report detection."""
        detector = DuplicateDetector()
        result = detector.check_duplicate({
            'chw_id': 'chw_001',
            'drug_id': 'amoxicillin',
            'pathogen_id': 'ecoli',
            'patient_age': 25,
        })
        assert result['is_duplicate'] == False  # No prior reports
    
    def test_quality_scoring(self):
        """Test report quality scoring."""
        scorer = ReportQualityScorer()
        full_report = {
            'drug_id': 'amoxicillin',
            'pathogen_id': 'ecoli',
            'patient_age': 30,
            'patient_sex': 'F',
            'resistance_status': 'susceptible',
            'treatment_given': 'yes',
            'outcome': 'recovered',
        }
        score = scorer.score(full_report)
        assert score == 1.0  # Perfect score
    
    def test_quality_multiplier(self):
        """Test quality-based point multiplier."""
        scorer = ReportQualityScorer()
        assert scorer.get_bonus_multiplier(0.95) == 1.0
        assert scorer.get_bonus_multiplier(0.75) == 0.8
        assert scorer.get_bonus_multiplier(0.3) == 0.0
    
    def test_leaderboard_ranking(self):
        """Test leaderboard computation."""
        engine = LeaderboardEngine(mock_db)
        result = await engine.update_ranking('chw_001')
        assert 'district_weekly' in result
    
    def test_point_decay(self):
        """Test point decay calculation."""
        engine = PointDecayEngine()
        decayed = engine.calculate_decayed_points(1000, 9)  # 9 months inactive
        # After 3 months of decay at 2%/month: 1000 * 0.98^3 ≈ 941
        assert 935 < decayed < 945
    
    def test_reward_redemption(self):
        """Test reward redemption flow."""
        service = RewardFulfillmentService()
        result = await service.redeem_reward('chw_001', 'airtime_2usd')
        assert result['status'] in ('fulfilled', 'pending')
```

---

## 14. Localization

| Element | Swahili | Yoruba | Hausa | French |
|---------|---------|--------|-------|--------|
| Points | Alama | Iwọn | Maki | Points |
| Badge | Beji | Bajo | Lada | Badge |
| Streak | Mfuatano | Ìtẹ̀síwájú | Jeri | Série |
| Leaderboard | Ubao wa Wafanya | Ibòjú | Teburin Shugabai | Classement |
| Reward | Tuzo | Ẹ̀bùn | Kyauta | Récompense |
| Tier | Daraja | Ìpele | Mataki | Niveau |
| Guardian | Mlinzi | Aṣàánú | Mai Gadi | Gardien |

---

## 15. Appendix

### A. Points Economy Design

The points economy is designed so that an average active CHW earns enough for meaningful monthly rewards:

```
AVERAGE ACTIVE CHW MONTHLY EARNINGS:
─────────────────────────────────────
Reports (5/week × 4 weeks × 10 pts)    = 200 pts
Voice bonus (60% voice × 3 pts × 20)    = 36 pts
Streak (daily 5pts × 25 days)          = 125 pts
Quality bonus (avg 80% × 5 pts × 20)   = 80 pts
First report of day (5 pts × 20)       = 100 pts
─────────────────────────────────────
TOTAL MONTHLY                         ≈ 541 pts
─────────────────────────────────────
TIER: Silver Guardian (100-500 range → topped out at Gold)
MONTHLY AIRTIME: $5 (250 pts) + keep 291 pts for catalog
```

### B. ROI Calculation Template

| Line Item | Value |
|-----------|-------|
| Annual gamification budget | $50,000 |
| Number of active CHWs | 500 |
| Cost per CHW per year | $100 |
| Reports per CHW per year (with gamification) | 420 |
| Reports per CHW per year (without gamification) | 150 |
| Additional reports attributable to gamification | 270 |
| Cost per additional report | $0.37 |
| Value per report (estimated based on early outbreak detection) | $5.00 |
| ROI | 13.5× |
| Net benefit | $625,000/year |

---

*Document generated as part of the UDARA AI Technical Documentation Series. For questions, contact the Product team at gamification@udara-ai.org.*

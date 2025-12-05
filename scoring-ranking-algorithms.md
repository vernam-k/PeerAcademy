# Peer Academy - Weighted Scoring & Ranking Algorithms

## Overview

The scoring and ranking system is the mathematical foundation of Peer Academy's meritocratic governance. It transforms peer evaluations into weighted voting power, determines representative selection, and maintains system integrity through sophisticated anti-gaming mechanisms. The algorithms must be transparent, fair, and resistant to manipulation while encouraging genuine academic excellence.

## Core Scoring Framework

### Mathematical Foundation
```typescript
interface ScoringParameters {
  // Base scoring weights
  evaluationWeight: number;        // Weight of peer evaluations (0.7)
  consistencyWeight: number;       // Reward for consistent performance (0.2)
  participationWeight: number;     // Reward for active evaluation (0.1)
  
  // Anti-gaming parameters
  decayRate: number;              // Historical score decay (0.05 per cycle)
  outlierThreshold: number;       // Standard deviations for outlier detection (2.0)
  minimumEvaluators: number;      // Minimum evaluators for valid score (3)
  maxVotingWeightRatio: number;   // Maximum voting weight ratio (10.0)
  
  // Normalization parameters
  subjectNormalization: boolean;   // Normalize scores within subject (true)
  globalNormalization: boolean;    // Apply global normalization (true)
  seasonalAdjustment: boolean;     // Adjust for seasonal variations (true)
}

const DEFAULT_SCORING_PARAMS: ScoringParameters = {
  evaluationWeight: 0.7,
  consistencyWeight: 0.2,
  participationWeight: 0.1,
  decayRate: 0.05,
  outlierThreshold: 2.0,
  minimumEvaluators: 3,
  maxVotingWeightRatio: 10.0,
  subjectNormalization: true,
  globalNormalization: true,
  seasonalAdjustment: true
};
```

### Core Scoring Algorithm
```typescript
class WeightedScoringEngine {
  private params: ScoringParameters;
  
  constructor(params: ScoringParameters = DEFAULT_SCORING_PARAMS) {
    this.params = params;
  }
  
  /**
   * Calculate the final weighted score for a presentation
   */
  async calculatePresentationScore(
    presentationId: string,
    evaluations: PresentationEvaluation[]
  ): Promise<ScoringResult> {
    
    // Step 1: Validate minimum requirements
    if (evaluations.length < this.params.minimumEvaluators) {
      return { score: 0, confidence: 0, issues: ['Insufficient evaluators'] };
    }
    
    // Step 2: Remove outlier evaluations
    const cleanedEvaluations = this.removeOutliers(evaluations);
    
    // Step 3: Calculate base weighted score
    const baseScore = this.calculateWeightedAverage(cleanedEvaluations);
    
    // Step 4: Apply quality adjustments
    const qualityMultiplier = this.calculateQualityMultiplier(cleanedEvaluations);
    
    // Step 5: Apply participation bonus
    const participationMultiplier = await this.calculateParticipationMultiplier(
      presentationId, 
      evaluations.length
    );
    
    // Step 6: Calculate final score
    const rawScore = baseScore * qualityMultiplier * participationMultiplier;
    
    // Step 7: Normalize score
    const normalizedScore = await this.normalizeScore(presentationId, rawScore);
    
    return {
      score: Math.max(0, Math.min(10, normalizedScore)),
      confidence: this.calculateConfidence(cleanedEvaluations),
      components: {
        baseScore,
        qualityMultiplier,
        participationMultiplier,
        normalizationFactor: normalizedScore / rawScore
      },
      evaluationsUsed: cleanedEvaluations.length,
      evaluationsRemoved: evaluations.length - cleanedEvaluations.length
    };
  }
  
  /**
   * Remove outlier evaluations using statistical methods
   */
  private removeOutliers(evaluations: PresentationEvaluation[]): PresentationEvaluation[] {
    if (evaluations.length < 5) return evaluations; // Need enough data for outlier detection
    
    const scores = evaluations.map(e => e.scores.overall);
    const mean = scores.reduce((a, b) => a + b) / scores.length;
    const stdDev = Math.sqrt(
      scores.map(s => Math.pow(s - mean, 2)).reduce((a, b) => a + b) / scores.length
    );
    
    const threshold = this.params.outlierThreshold * stdDev;
    
    return evaluations.filter(eval => 
      Math.abs(eval.scores.overall - mean) <= threshold
    );
  }
  
  /**
   * Calculate weighted average considering evaluator voting weights
   */
  private calculateWeightedAverage(evaluations: PresentationEvaluation[]): number {
    const totalWeight = evaluations.reduce((sum, eval) => sum + eval.votingWeight, 0);
    const weightedSum = evaluations.reduce((sum, eval) => {
      return sum + (eval.scores.overall * eval.votingWeight);
    }, 0);
    
    return weightedSum / totalWeight;
  }
  
  /**
   * Calculate quality multiplier based on score consistency across categories
   */
  private calculateQualityMultiplier(evaluations: PresentationEvaluation[]): number {
    // Calculate average scores across all categories
    const categoryAverages = this.calculateCategoryAverages(evaluations);
    
    // Reward consistency across categories (less variance = higher quality)
    const scores = Object.values(categoryAverages);
    const variance = this.calculateVariance(scores);
    
    // Convert variance to a multiplier (lower variance = higher multiplier)
    const consistencyBonus = Math.max(0, 1 - (variance / 5)); // Max variance penalty of 1.0
    
    return 1 + (consistencyBonus * 0.2); // Up to 20% bonus for consistency
  }
  
  private calculateCategoryAverages(evaluations: PresentationEvaluation[]): CategoryScores {
    const categories = ['content', 'clarity', 'originality', 'relevance', 'engagement'];
    const averages: CategoryScores = {};
    
    categories.forEach(category => {
      const totalWeight = evaluations.reduce((sum, eval) => sum + eval.votingWeight, 0);
      const weightedSum = evaluations.reduce((sum, eval) => {
        return sum + (eval.scores[category] * eval.votingWeight);
      }, 0);
      averages[category] = weightedSum / totalWeight;
    });
    
    return averages;
  }
}

interface ScoringResult {
  score: number;
  confidence: number;
  components?: {
    baseScore: number;
    qualityMultiplier: number;
    participationMultiplier: number;
    normalizationFactor: number;
  };
  evaluationsUsed: number;
  evaluationsRemoved: number;
  issues?: string[];
}

interface CategoryScores {
  [category: string]: number;
}
```

### Cumulative Score Calculation
```typescript
class CumulativeScoreEngine {
  /**
   * Update member's cumulative score with new presentation score
   */
  async updateCumulativeScore(
    userId: string,
    subjectId: string,
    newScore: number,
    cycleNumber: number
  ): Promise<CumulativeScoreResult> {
    
    const member = await this.getSubjectMembership(userId, subjectId);
    const historicalScores = await this.getHistoricalScores(userId, subjectId);
    
    // Add new score to history
    const newWeeklyScore: WeeklyScore = {
      cycleNumber,
      score: newScore,
      timestamp: new Date(),
      weight: 1.0 // Full weight for current score
    };
    
    historicalScores.push(newWeeklyScore);
    
    // Apply time decay to historical scores
    const decayedScores = this.applyTimeDecay(historicalScores, cycleNumber);
    
    // Calculate new cumulative score
    const cumulativeScore = this.calculateCumulativeScore(decayedScores);
    
    // Update member record
    await this.updateMemberScore(userId, subjectId, cumulativeScore, newWeeklyScore);
    
    // Update voting weight
    const newVotingWeight = this.calculateVotingWeight(cumulativeScore, subjectId);
    await this.updateVotingWeight(userId, subjectId, newVotingWeight);
    
    return {
      newCumulativeScore: cumulativeScore,
      newVotingWeight,
      rank: await this.calculateSubjectRank(userId, subjectId),
      percentile: await this.calculatePercentile(userId, subjectId)
    };
  }
  
  /**
   * Apply time decay to historical scores
   */
  private applyTimeDecay(scores: WeeklyScore[], currentCycle: number): WeeklyScore[] {
    return scores.map(score => {
      const ageInCycles = currentCycle - score.cycleNumber;
      const decayFactor = Math.pow(1 - this.params.decayRate, ageInCycles);
      
      return {
        ...score,
        weight: score.weight * decayFactor
      };
    });
  }
  
  /**
   * Calculate cumulative score from weighted historical scores
   */
  private calculateCumulativeScore(weightedScores: WeeklyScore[]): number {
    if (weightedScores.length === 0) return 0;
    
    const totalWeight = weightedScores.reduce((sum, score) => sum + score.weight, 0);
    const weightedSum = weightedScores.reduce((sum, score) => 
      sum + (score.score * score.weight), 0
    );
    
    return weightedSum / totalWeight;
  }
  
  /**
   * Calculate voting weight from cumulative score
   * Uses logarithmic scale to prevent extreme concentration of power
   */
  calculateVotingWeight(cumulativeScore: number, subjectId?: string): number {
    // Base voting weight using logarithmic scale
    const baseWeight = Math.log(Math.max(1, cumulativeScore + 1));
    
    // Apply subject-specific normalization
    const subjectMultiplier = this.getSubjectMultiplier(subjectId);
    
    // Cap maximum voting weight to prevent extreme concentration
    const cappedWeight = Math.min(
      baseWeight * subjectMultiplier,
      this.params.maxVotingWeightRatio
    );
    
    // Ensure minimum voting weight of 1
    return Math.max(1, cappedWeight);
  }
  
  private getSubjectMultiplier(subjectId?: string): number {
    // Different subjects may have different scoring scales
    // This normalizes voting power across subjects
    if (!subjectId) return 1.0;
    
    // In practice, this would be calculated from historical data
    // For now, return base multiplier
    return 1.0;
  }
}

interface CumulativeScoreResult {
  newCumulativeScore: number;
  newVotingWeight: number;
  rank: number;
  percentile: number;
}

interface WeeklyScore {
  cycleNumber: number;
  score: number;
  timestamp: Date;
  weight: number;
}
```

## Representative Selection Algorithm

### Merit-Based Selection
```typescript
class RepresentativeSelectionEngine {
  /**
   * Select representative for subject based on comprehensive merit evaluation
   */
  async selectRepresentative(subjectId: string, cycleNumber: number): Promise<RepresentativeSelection> {
    const activeMembers = await this.getActiveMembers(subjectId);
    
    if (activeMembers.length === 0) {
      throw new Error('No active members to select representative from');
    }
    
    // Calculate comprehensive merit scores for each member
    const meritScores = await Promise.all(
      activeMembers.map(member => this.calculateMeritScore(member, cycleNumber))
    );
    
    // Sort by merit score
    const sortedCandidates = meritScores
      .sort((a, b) => b.totalMerit - a.totalMerit);
    
    // Select top candidate with additional validation
    const selectedCandidate = await this.validateAndSelectCandidate(
      sortedCandidates, 
      subjectId, 
      cycleNumber
    );
    
    // Record selection decision
    await this.recordRepresentativeSelection(subjectId, selectedCandidate, cycleNumber);
    
    return {
      selectedUserId: selectedCandidate.userId,
      meritScore: selectedCandidate.totalMerit,
      selectionReason: selectedCandidate.selectionReason,
      allCandidates: sortedCandidates.slice(0, 5), // Top 5 for transparency
      effectiveDate: new Date(),
      termLength: '1 cycle' // Representatives serve one cycle
    };
  }
  
  /**
   * Calculate comprehensive merit score considering multiple factors
   */
  private async calculateMeritScore(
    member: SubjectMembership, 
    cycleNumber: number
  ): Promise<MeritCandidate> {
    
    // Core academic performance (60% weight)
    const academicScore = member.cumulativeScore;
    
    // Recent performance trend (20% weight)
    const trendScore = await this.calculateTrendScore(member.userId, member.subjectId);
    
    // Participation and engagement (15% weight)
    const participationScore = await this.calculateParticipationScore(
      member.userId, 
      member.subjectId
    );
    
    // Leadership experience (5% weight)
    const leadershipScore = await this.calculateLeadershipScore(member.userId);
    
    // Calculate weighted total merit
    const totalMerit = (
      academicScore * 0.60 +
      trendScore * 0.20 +
      participationScore * 0.15 +
      leadershipScore * 0.05
    );
    
    return {
      userId: member.userId,
      academicScore,
      trendScore,
      participationScore,
      leadershipScore,
      totalMerit,
      eligibilityIssues: await this.checkEligibility(member.userId, member.subjectId)
    };
  }
  
  /**
   * Calculate performance trend over recent cycles
   */
  private async calculateTrendScore(userId: string, subjectId: string): Promise<number> {
    const recentScores = await this.getRecentScores(userId, subjectId, 5); // Last 5 cycles
    
    if (recentScores.length < 2) return 0;
    
    // Calculate linear regression slope
    const slope = this.calculateTrendSlope(recentScores);
    
    // Convert slope to score (positive trend = higher score)
    // Normalize slope to 0-10 scale
    return Math.max(0, Math.min(10, 5 + slope * 2));
  }
  
  private calculateTrendSlope(scores: WeeklyScore[]): number {
    const n = scores.length;
    if (n < 2) return 0;
    
    const xSum = scores.reduce((sum, _, index) => sum + index, 0);
    const ySum = scores.reduce((sum, score) => sum + score.score, 0);
    const xySum = scores.reduce((sum, score, index) => sum + (index * score.score), 0);
    const x2Sum = scores.reduce((sum, _, index) => sum + (index * index), 0);
    
    const slope = (n * xySum - xSum * ySum) / (n * x2Sum - xSum * xSum);
    return slope;
  }
  
  /**
   * Validate candidate eligibility and select best qualified
   */
  private async validateAndSelectCandidate(
    candidates: MeritCandidate[],
    subjectId: string,
    cycleNumber: number
  ): Promise<MeritCandidate> {
    
    for (const candidate of candidates) {
      // Check basic eligibility
      if (candidate.eligibilityIssues.length > 0) {
        candidate.selectionReason = `Ineligible: ${candidate.eligibilityIssues.join(', ')}`;
        continue;
      }
      
      // Check minimum merit threshold
      if (candidate.totalMerit < 5.0) {
        candidate.selectionReason = 'Below minimum merit threshold';
        continue;
      }
      
      // Check availability and willingness
      const isAvailable = await this.checkAvailability(candidate.userId);
      if (!isAvailable) {
        candidate.selectionReason = 'Not available for representative role';
        continue;
      }
      
      // This candidate passes all checks
      candidate.selectionReason = 'Selected based on highest merit score and eligibility';
      return candidate;
    }
    
    throw new Error('No eligible candidates found for representative role');
  }
  
  private async checkEligibility(userId: string, subjectId: string): Promise<string[]> {
    const issues: string[] = [];
    
    // Check minimum membership duration
    const membership = await this.getSubjectMembership(userId, subjectId);
    const membershipWeeks = this.calculateWeeksSince(membership.joinedAt);
    
    if (membershipWeeks < 4) {
      issues.push('Insufficient membership duration (minimum 4 weeks)');
    }
    
    // Check minimum participation
    const participationRate = await this.getParticipationRate(userId, subjectId);
    if (participationRate < 0.75) {
      issues.push('Low participation rate (minimum 75%)');
    }
    
    // Check current sanctions or restrictions
    const hasSanctions = await this.hasSanctions(userId);
    if (hasSanctions) {
      issues.push('Active sanctions or restrictions');
    }
    
    return issues;
  }
}

interface MeritCandidate {
  userId: string;
  academicScore: number;
  trendScore: number;
  participationScore: number;
  leadershipScore: number;
  totalMerit: number;
  eligibilityIssues: string[];
  selectionReason?: string;
}

interface RepresentativeSelection {
  selectedUserId: string;
  meritScore: number;
  selectionReason: string;
  allCandidates: MeritCandidate[];
  effectiveDate: Date;
  termLength: string;
}
```

## Anti-Gaming Mechanisms

### Gaming Detection System
```typescript
class AntiGamingSystem {
  private suspicionThreshold = 0.7; // 0-1 scale
  
  /**
   * Detect potential gaming behavior in evaluations
   */
  async detectGamingBehavior(
    evaluations: PresentationEvaluation[],
    presentationId: string
  ): Promise<GamingDetectionResult> {
    
    const detectionResults = await Promise.all([
      this.detectCollusionPatterns(evaluations),
      this.detectBiasPatterns(evaluations),
      this.detectManipulationAttempts(evaluations),
      this.detectSybilAttacks(evaluations)
    ]);
    
    const maxSuspicionLevel = Math.max(...detectionResults.map(r => r.suspicionLevel));
    const allIssues = detectionResults.flatMap(r => r.issues);
    
    const result: GamingDetectionResult = {
      presentationId,
      suspicionLevel: maxSuspicionLevel,
      issues: allIssues,
      requiresReview: maxSuspicionLevel > this.suspicionThreshold,
      detectionDetails: detectionResults
    };
    
    if (result.requiresReview) {
      await this.flagForManualReview(presentationId, result);
    }
    
    return result;
  }
  
  /**
   * Detect collusion patterns among evaluators
   */
  private async detectCollusionPatterns(
    evaluations: PresentationEvaluation[]
  ): Promise<DetectionResult> {
    
    const issues: string[] = [];
    let suspicionLevel = 0;
    
    // Check for identical score patterns
    const scorePatterns = this.extractScorePatterns(evaluations);
    const identicalPatterns = this.findIdenticalPatterns(scorePatterns);
    
    if (identicalPatterns.length > 1) {
      issues.push(`${identicalPatterns.length} evaluators with identical score patterns`);
      suspicionLevel = Math.max(suspicionLevel, 0.6);
    }
    
    // Check for unusual timing patterns
    const timingPatterns = await this.analyzeTimingPatterns(evaluations);
    if (timingPatterns.suspiciouslyClose > 2) {
      issues.push('Multiple evaluations submitted within suspicious timeframe');
      suspicionLevel = Math.max(suspicionLevel, 0.5);
    }
    
    // Check for reciprocal scoring patterns
    const reciprocalPatterns = await this.detectReciprocalScoring(evaluations);
    if (reciprocalPatterns.count > 0) {
      issues.push('Potential reciprocal scoring detected');
      suspicionLevel = Math.max(suspicionLevel, 0.7);
    }
    
    return {
      type: 'collusion',
      suspicionLevel,
      issues,
      evidence: {
        identicalPatterns,
        timingPatterns,
        reciprocalPatterns
      }
    };
  }
  
  /**
   * Detect systematic bias patterns
   */
  private async detectBiasPatterns(
    evaluations: PresentationEvaluation[]
  ): Promise<DetectionResult> {
    
    const issues: string[] = [];
    let suspicionLevel = 0;
    
    // Check for extreme scoring patterns
    const extremeScores = evaluations.filter(eval => 
      eval.scores.overall <= 2 || eval.scores.overall >= 9
    );
    
    if (extremeScores.length / evaluations.length > 0.5) {
      issues.push('Unusually high proportion of extreme scores');
      suspicionLevel = Math.max(suspicionLevel, 0.4);
    }
    
    // Check for demographic bias indicators
    const biasIndicators = await this.checkDemographicBias(evaluations);
    if (biasIndicators.length > 0) {
      issues.push(...biasIndicators);
      suspicionLevel = Math.max(suspicionLevel, 0.6);
    }
    
    return {
      type: 'bias',
      suspicionLevel,
      issues,
      evidence: {
        extremeScoreRatio: extremeScores.length / evaluations.length,
        biasIndicators
      }
    };
  }
  
  /**
   * Detect manipulation attempts
   */
  private async detectManipulationAttempts(
    evaluations: PresentationEvaluation[]
  ): Promise<DetectionResult> {
    
    const issues: string[] = [];
    let suspicionLevel = 0;
    
    // Check for rapid-fire evaluations (too fast to properly review)
    const rapidEvaluations = evaluations.filter(eval => 
      eval.timeSpent && eval.timeSpent < 5 // Less than 5 minutes
    );
    
    if (rapidEvaluations.length > evaluations.length * 0.3) {
      issues.push('Unusually fast evaluation times detected');
      suspicionLevel = Math.max(suspicionLevel, 0.5);
    }
    
    // Check for duplicate IP addresses or browser fingerprints
    const duplicateOrigins = await this.detectDuplicateOrigins(evaluations);
    if (duplicateOrigins.count > 0) {
      issues.push('Multiple evaluations from same origin detected');
      suspicionLevel = Math.max(suspicionLevel, 0.8);
    }
    
    return {
      type: 'manipulation',
      suspicionLevel,
      issues,
      evidence: {
        rapidEvaluations: rapidEvaluations.length,
        duplicateOrigins
      }
    };
  }
  
  /**
   * Apply penalties for detected gaming behavior
   */
  async applyGamingPenalties(
    detection: GamingDetectionResult,
    affectedUserIds: string[]
  ): Promise<void> {
    
    const penaltyLevel = this.calculatePenaltyLevel(detection.suspicionLevel);
    
    for (const userId of affectedUserIds) {
      const penalty: GamingPenalty = {
        userId,
        type: this.determinePenaltyType(detection),
        severity: penaltyLevel,
        duration: this.calculatePenaltyDuration(penaltyLevel),
        reason: detection.issues.join('; '),
        appliedAt: new Date()
      };
      
      await this.applyPenalty(penalty);
    }
  }
  
  private calculatePenaltyLevel(suspicionLevel: number): PenaltySeverity {
    if (suspicionLevel >= 0.9) return PenaltySeverity.SEVERE;
    if (suspicionLevel >= 0.7) return PenaltySeverity.MODERATE;
    if (suspicionLevel >= 0.5) return PenaltySeverity.MINOR;
    return PenaltySeverity.WARNING;
  }
}

interface GamingDetectionResult {
  presentationId: string;
  suspicionLevel: number;
  issues: string[];
  requiresReview: boolean;
  detectionDetails: DetectionResult[];
}

interface DetectionResult {
  type: 'collusion' | 'bias' | 'manipulation' | 'sybil';
  suspicionLevel: number;
  issues: string[];
  evidence: any;
}

enum PenaltySeverity {
  WARNING = 'warning',
  MINOR = 'minor',
  MODERATE = 'moderate',
  SEVERE = 'severe'
}
```

## Performance Analytics & Insights

### Advanced Analytics Engine
```typescript
class PerformanceAnalyticsEngine {
  /**
   * Generate comprehensive performance analytics for a member
   */
  async generateMemberAnalytics(
    userId: string,
    subjectId: string,
    timeRange: TimeRange
  ): Promise<MemberAnalytics> {
    
    const [
      performanceHistory,
      peerComparison,
      strengthsWeaknesses,
      trendAnalysis,
      participationMetrics
    ] = await Promise.all([
      this.getPerformanceHistory(userId, subjectId, timeRange),
      this.generatePeerComparison(userId, subjectId),
      this.analyzeStrengthsWeaknesses(userId, subjectId),
      this.analyzeTrends(userId, subjectId, timeRange),
      this.calculateParticipationMetrics(userId, subjectId)
    ]);
    
    return {
      userId,
      subjectId,
      timeRange,
      performanceHistory,
      peerComparison,
      strengthsWeaknesses,
      trendAnalysis,
      participationMetrics,
      recommendations: await this.generateRecommendations(userId, subjectId),
      generatedAt: new Date()
    };
  }
  
  /**
   * Analyze member strengths and weaknesses across evaluation categories
   */
  private async analyzeStrengthsWeaknesses(
    userId: string,
    subjectId: string
  ): Promise<StrengthsWeaknessesAnalysis> {
    
    const evaluations = await this.getMemberEvaluations(userId, subjectId);
    const categories = ['content', 'clarity', 'originality', 'relevance', 'engagement'];
    
    const categoryPerformance = categories.map(category => {
      const scores = evaluations.map(eval => eval.scores[category]);
      const average = scores.reduce((a, b) => a + b, 0) / scores.length;
      const consistency = 1 - this.calculateVariance(scores) / 10; // 0-1 scale
      
      return {
        category,
        average,
        consistency,
        trend: this.calculateCategoryTrend(evaluations, category),
        percentile: this.calculateCategoryPercentile(userId, subjectId, category)
      };
    });
    
    // Identify top strengths and areas for improvement
    const sortedByPerformance = [...categoryPerformance]
      .sort((a, b) => b.average - a.average);
    
    return {
      strengths: sortedByPerformance.slice(0, 2),
      weaknesses: sortedByPerformance.slice(-2),
      mostImproved: sortedByPerformance
        .sort((a, b) => b.trend - a.trend)[0],
      mostConsistent: sortedByPerformance
        .sort((a, b) => b.consistency - a.consistency)[0]
    };
  }
  
  /**
   * Generate personalized improvement recommendations
   */
  private async generateRecommendations(
    userId: string,
    subjectId: string
  ): Promise<Recommendation[]> {
    
    const analytics = await this.getMemberAnalytics(userId, subjectId);
    const recommendations: Recommendation[] = [];
    
    // Content-based recommendations
    if (analytics.strengthsWeaknesses.weaknesses.some(w => w.category === 'content')) {
      recommendations.push({
        type: 'improvement',
        category: 'content',
        priority: 'high',
        title: 'Enhance Content Depth',
        description: 'Focus on providing more comprehensive and well-researched content',
        actionItems: [
          'Research topics more thoroughly before presenting',
          'Include multiple perspectives and sources',
          'Provide concrete examples and case studies'
        ]
      });
    }
    
    // Participation recommendations
    if (analytics.participationMetrics.evaluationRate < 0.8) {
      recommendations.push({
        type: 'engagement',
        category: 'participation',
        priority: 'medium',
        title: 'Increase Peer Evaluation Participation',
        description: 'Actively evaluate more peer presentations to improve your voting weight',
        actionItems: [
          'Set aside time weekly for peer evaluations',
          'Provide thoughtful feedback to help peers improve',
          'Participate in live evaluation sessions'
        ]
      });
    }
    
    return recommendations;
  }
}

interface MemberAnalytics {
  userId: string;
  subjectId: string;
  timeRange: TimeRange;
  performanceHistory: PerformancePoint[];
  peerComparison: PeerComparison;
  strengthsWeaknesses: StrengthsWeaknessesAnalysis;
  trendAnalysis: TrendAnalysis;
  participationMetrics: ParticipationMetrics;
  recommendations: Recommendation[];
  generatedAt: Date;
}

interface Recommendation {
  type: 'improvement' | 'enhancement' | 'engagement';
  category: string;
  priority: 'low' | 'medium' | 'high';
  title: string;
  description: string;
  actionItems: string[];
}
```

## Database Schema

### Scoring Tables
```sql
-- Member scores and history
CREATE TABLE member_scores (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  cycle_number INTEGER NOT NULL,
  presentation_score DECIMAL(4,2),
  cumulative_score DECIMAL(8,2) DEFAULT 0,
  voting_weight DECIMAL(8,2) DEFAULT 1,
  rank_in_subject INTEGER,
  percentile DECIMAL(5,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(user_id, subject_id, cycle_number)
);

-- Representative selections
CREATE TABLE representative_selections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  cycle_number INTEGER NOT NULL,
  merit_score DECIMAL(6,2) NOT NULL,
  selection_reason TEXT,
  effective_date TIMESTAMP NOT NULL,
  term_length VARCHAR(50),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  
  UNIQUE(subject_id, cycle_number)
);

-- Gaming detection logs
CREATE TABLE gaming_detections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  presentation_id UUID REFERENCES presentations(id) ON DELETE CASCADE,
  detection_type VARCHAR(50) NOT NULL,
  suspicion_level DECIMAL(3,2) NOT NULL,
  issues JSONB NOT NULL,
  evidence JSONB,
  requires_review BOOLEAN DEFAULT FALSE,
  reviewed_at TIMESTAMP,
  reviewed_by UUID REFERENCES users(id),
  action_taken VARCHAR(100),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Gaming penalties
CREATE TABLE gaming_penalties (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  penalty_type VARCHAR(50) NOT NULL,
  severity VARCHAR(20) NOT NULL,
  duration_days INTEGER,
  reason TEXT NOT NULL,
  applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP,
  revoked_at TIMESTAMP,
  revoked_by UUID REFERENCES users(id)
);

-- Performance analytics cache
CREATE TABLE performance_analytics (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  subject_id UUID REFERENCES subjects(id) ON DELETE CASCADE,
  analytics_data JSONB NOT NULL,
  time_range VARCHAR(50),
  generated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
  expires_at TIMESTAMP DEFAULT (CURRENT_TIMESTAMP + INTERVAL '1 day')
);

-- Indexes for performance
CREATE INDEX idx_member_scores_user_subject ON member_scores(user_id, subject_id);
CREATE INDEX idx_member_scores_cumulative ON member_scores(cumulative_score DESC);
CREATE INDEX idx_member_scores_cycle ON member_scores(cycle_number);
CREATE INDEX idx_representative_selections_subject ON representative_selections(subject_id);
CREATE INDEX idx_gaming_detections_presentation ON gaming_detections(presentation_id);
CREATE INDEX idx_gaming_penalties_user ON gaming_penalties(user_id);
CREATE INDEX idx_performance_analytics_user ON performance_analytics(user_id, subject_id);
```

This weighted scoring and ranking system provides the mathematical foundation for Peer Academy's merit-based governance, with sophisticated algorithms that reward excellence while preventing gaming and maintaining fairness across the platform.
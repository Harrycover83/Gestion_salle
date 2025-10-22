# GymManager Pro - Spécifications Techniques Détaillées

## Application Web pour Gérants de Salles de Sport

### Table des Matières
1. [Architecture Technique](#architecture-technique)
2. [Modules Fonctionnels](#modules-fonctionnels)
3. [API Endpoints](#api-endpoints)
4. [Modèles de Données](#modèles-de-données)
5. [Sécurité et Conformité](#sécurité-et-conformité)
6. [Intégrations](#intégrations)
7. [Performance et Scalabilité](#performance-et-scalabilité)

---

## Architecture Technique

### Stack Technologique Recommandé

#### Frontend
```
Framework: React 18+ avec TypeScript
├── UI Library: Material-UI (MUI) v5
├── State Management: Redux Toolkit + RTK Query
├── Routing: React Router v6
├── Forms: React Hook Form + Zod validation
├── Charts: Recharts + ApexCharts
├── Date/Time: date-fns
├── i18n: react-i18next
└── Build Tool: Vite
```

#### Backend
```
Framework: Node.js 20+ avec TypeScript
├── API Framework: NestJS 10+
├── ORM: Prisma
├── Authentication: Passport.js + JWT
├── Validation: class-validator
├── Documentation: Swagger/OpenAPI
├── Queue: Bull (Redis)
└── Caching: Redis
```

#### Base de Données
```
Principale: PostgreSQL 15+
├── Schéma: Multi-tenant avec row-level security
├── Extensions: pg_trgm (full-text search), pgcrypto
└── Backup: Point-in-time recovery activé

Secondaires:
├── Redis: Cache, sessions, queues
├── ElasticSearch: Recherche full-text avancée
└── S3-compatible: Documents et médias
```

### Architecture des Services

```
services/
├── api-gateway/
│   ├── Authentication & authorization
│   ├── Rate limiting
│   ├── Request routing
│   └── Load balancing
├── member-service/
│   ├── CRUD membres
│   ├── Abonnements
│   ├── Documents membres
│   └── Contrôle d'accès
├── scheduling-service/
│   ├── Planning cours
│   ├── Réservations
│   ├── Disponibilités
│   └── Conflits detection
├── payment-service/
│   ├── Facturation
│   ├── Paiements (Stripe)
│   ├── Prélèvements SEPA
│   └── Gestion impayés
├── communication-service/
│   ├── Emails (SendGrid)
│   ├── SMS (Twilio)
│   ├── Push notifications
│   └── Templates
├── analytics-service/
│   ├── Data warehouse
│   ├── Reporting
│   ├── KPI calculations
│   └── ML predictions
└── document-service/
    ├── Upload sécurisé
    ├── Chiffrement
    ├── OCR
    └── Archivage
```

---

## Modules Fonctionnels

### 1. Module Gestion Membres

#### Fonctionnalités Détaillées

##### 1.1 Fiche Membre Complète
```typescript
interface Member {
  // Identification
  id: string;
  memberNumber: string; // Auto-généré, unique
  externalId?: string; // ID système tiers
  
  // Informations personnelles (RGPD)
  firstName: string;
  lastName: string;
  email: string;
  phone: string;
  dateOfBirth: Date;
  gender?: 'M' | 'F' | 'Other' | 'PreferNotToSay';
  photo?: string; // URL sécurisée
  
  // Adresse
  address: {
    street: string;
    city: string;
    postalCode: string;
    country: string;
  };
  
  // Contact d'urgence
  emergencyContact: {
    name: string;
    relationship: string;
    phone: string;
  };
  
  // Informations médicales (chiffrées)
  medicalInfo?: {
    medicalCertificate?: {
      url: string; // Chiffré
      expiryDate: Date;
      uploadDate: Date;
      verified: boolean;
    };
    restrictions?: string[];
    allergies?: string[];
    medications?: string[];
    conditions?: string[];
  };
  
  // Abonnement actuel
  currentSubscription?: {
    id: string;
    type: string;
    startDate: Date;
    endDate: Date;
    status: 'active' | 'suspended' | 'expired' | 'cancelled';
    autoRenew: boolean;
  };
  
  // Historique
  memberSince: Date;
  lastVisit?: Date;
  totalVisits: number;
  
  // Préférences
  preferences: {
    language: string;
    notifications: {
      email: boolean;
      sms: boolean;
      push: boolean;
    };
    marketingConsent: boolean;
  };
  
  // Métadonnées
  tags: string[];
  notes: string;
  status: 'active' | 'inactive' | 'suspended' | 'banned';
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}
```

##### 1.2 Gestion des Abonnements
```typescript
interface Subscription {
  id: string;
  memberId: string;
  
  // Type et tarification
  subscriptionType: {
    id: string;
    name: string;
    description: string;
    price: number;
    currency: string;
    billingPeriod: 'monthly' | 'quarterly' | 'yearly';
  };
  
  // Dates
  startDate: Date;
  endDate: Date;
  trialEndDate?: Date;
  suspendedFrom?: Date;
  suspendedUntil?: Date;
  
  // Statut
  status: 'trial' | 'active' | 'suspended' | 'expired' | 'cancelled';
  autoRenew: boolean;
  cancellationReason?: string;
  
  // Accès
  accessRights: {
    facilities: string[]; // IDs des espaces accessibles
    classes: string[]; // Types de cours inclus
    hours: {
      monday: { start: string; end: string }[];
      // ... autres jours
    };
    guestPass: number; // Nombre d'invités permis
  };
  
  // Paiement
  paymentMethod: {
    type: 'card' | 'sepa' | 'cash' | 'check' | 'bank_transfer';
    lastFour?: string;
    expiryDate?: string;
  };
  nextBillingDate?: Date;
  
  // Historique
  createdAt: Date;
  updatedAt: Date;
  renewalHistory: Array<{
    date: Date;
    previousEndDate: Date;
    newEndDate: Date;
    amount: number;
  }>;
}

interface SubscriptionType {
  id: string;
  gymId: string;
  
  // Identification
  name: string;
  description: string;
  category: 'basic' | 'premium' | 'vip' | 'corporate' | 'student';
  
  // Tarification
  pricing: {
    basePrice: number;
    currency: string;
    billingPeriod: 'daily' | 'weekly' | 'monthly' | 'quarterly' | 'yearly';
    setupFee?: number;
    depositRequired?: number;
  };
  
  // Promotions
  discounts?: Array<{
    name: string;
    type: 'percentage' | 'fixed';
    value: number;
    validFrom: Date;
    validUntil: Date;
    conditions?: string;
  }>;
  
  // Accès et limites
  benefits: {
    accessLevel: 'unlimited' | 'limited';
    visitLimit?: number; // per period
    facilities: string[];
    classTypes: string[];
    hours: 'anytime' | 'off_peak' | 'custom';
    guestPassIncluded: number;
    personalTrainingSessions?: number;
    lockerIncluded: boolean;
    parkingIncluded: boolean;
  };
  
  // Règles
  rules: {
    minCommitment?: number; // mois
    cancellationNoticePeriod: number; // jours
    freezeAllowed: boolean;
    maxFreezeDays?: number; // per year
    transferable: boolean;
  };
  
  // Métadonnées
  isActive: boolean;
  isPublic: boolean; // visible sur site web
  maxMembers?: number; // capacité
  currentMembers: number;
  displayOrder: number;
  
  createdAt: Date;
  updatedAt: Date;
}
```

##### 1.3 Contrôle d'Accès
```typescript
interface AccessLog {
  id: string;
  memberId: string;
  gymId: string;
  facilityId: string;
  
  // Accès
  entryTime: Date;
  exitTime?: Date;
  accessMethod: 'card' | 'qr_code' | 'biometric' | 'manual' | 'app';
  accessPoint: string; // Nom de la porte/tourniquet
  
  // Validation
  status: 'granted' | 'denied' | 'warning';
  denialReason?: 'expired_subscription' | 'suspended' | 'wrong_hours' | 'capacity_full' | 'blocked';
  
  // Métadonnées
  deviceId?: string;
  ipAddress?: string;
  location?: {
    latitude: number;
    longitude: number;
  };
  
  createdAt: Date;
}

interface AccessDevice {
  id: string;
  gymId: string;
  
  name: string;
  type: 'turnstile' | 'door' | 'tablet' | 'mobile';
  location: string;
  
  config: {
    allowManualEntry: boolean;
    requirePhoto: boolean;
    alertOnDenial: boolean;
  };
  
  status: 'online' | 'offline' | 'maintenance';
  lastHeartbeat: Date;
  
  createdAt: Date;
  updatedAt: Date;
}
```

### 2. Module Planning et Réservations

#### 2.1 Cours Collectifs
```typescript
interface Class {
  id: string;
  gymId: string;
  
  // Informations de base
  name: string;
  description: string;
  type: string; // 'yoga', 'spinning', 'crossfit', etc.
  category: string; // 'cardio', 'strength', 'flexibility', etc.
  level: 'beginner' | 'intermediate' | 'advanced' | 'all_levels';
  
  // Planning
  schedule: {
    dayOfWeek: number; // 0-6 (dimanche-samedi)
    startTime: string; // HH:mm
    endTime: string; // HH:mm
    timezone: string;
  };
  
  // Dates de validité
  startDate: Date;
  endDate?: Date;
  
  // Exceptions (jours fériés, fermetures)
  excludedDates: Date[];
  
  // Capacité
  capacity: {
    min: number;
    max: number;
    waitlistMax?: number;
  };
  
  // Lieu et équipement
  facility: {
    roomId: string;
    roomName: string;
    equipment: string[]; // Matériel requis
  };
  
  // Instructeur
  instructor: {
    id: string;
    name: string;
    photo?: string;
  };
  substituteInstructor?: {
    id: string;
    name: string;
  };
  
  // Réservation
  bookingSettings: {
    requiresReservation: boolean;
    advanceBookingDays: number; // Combien de jours à l'avance
    cancellationDeadlineHours: number;
    lateBookingMinutes: number; // Minutes avant le cours
    autoReleaseMinutes: number; // Auto-libération si absent
  };
  
  // Tarification
  pricing?: {
    included: boolean; // Inclus dans abonnement
    dropInPrice?: number;
    packageSessions?: Array<{
      sessions: number;
      price: number;
      validityDays: number;
    }>;
  };
  
  // État
  status: 'scheduled' | 'cancelled' | 'completed' | 'in_progress';
  cancellationReason?: string;
  
  // Métadonnées
  isRecurring: boolean;
  parentClassId?: string; // Pour les instances d'un cours récurrent
  tags: string[];
  imageUrl?: string;
  
  createdAt: Date;
  updatedAt: Date;
}

interface ClassInstance {
  id: string;
  classId: string;
  
  // Date et heure spécifique
  date: Date;
  startTime: Date;
  endTime: Date;
  
  // Instructor (peut être différent du cours régulier)
  instructorId: string;
  
  // État des réservations
  bookings: {
    confirmed: number;
    waitlist: number;
    attended: number;
    noShows: number;
  };
  
  // Statut
  status: 'scheduled' | 'cancelled' | 'completed' | 'in_progress';
  
  // Notes
  notes?: string;
  
  createdAt: Date;
  updatedAt: Date;
}

interface Booking {
  id: string;
  memberId: string;
  classInstanceId: string;
  
  // Statut
  status: 'confirmed' | 'waitlist' | 'cancelled' | 'attended' | 'no_show';
  
  // Dates importantes
  bookedAt: Date;
  cancelledAt?: Date;
  attendedAt?: Date;
  
  // Paiement (si applicable)
  paymentId?: string;
  amount?: number;
  
  // Notes
  memberNotes?: string;
  staffNotes?: string;
  
  createdAt: Date;
  updatedAt: Date;
}
```

#### 2.2 Créneaux Personnels (PT, Nutrition, etc.)
```typescript
interface PersonalSession {
  id: string;
  gymId: string;
  memberId: string;
  
  // Type de session
  sessionType: 'personal_training' | 'nutrition_consultation' | 'physiotherapy' | 'wellness_coaching' | 'other';
  
  // Planning
  date: Date;
  startTime: Date;
  endTime: Date;
  duration: number; // minutes
  
  // Professionnel
  professional: {
    id: string;
    name: string;
    specialization: string;
  };
  
  // Lieu
  location: {
    roomId?: string;
    roomName?: string;
    isOnline: boolean;
    meetingLink?: string;
  };
  
  // Status
  status: 'scheduled' | 'confirmed' | 'cancelled' | 'completed' | 'no_show';
  cancellationReason?: string;
  
  // Notes
  sessionNotes?: string;
  memberGoals?: string[];
  nextSessionPlan?: string;
  
  // Facturation
  pricing: {
    amount: number;
    currency: string;
    paymentStatus: 'pending' | 'paid' | 'refunded';
    includedInPackage: boolean;
  };
  
  // Rappels
  reminders: {
    sent: boolean;
    sentAt?: Date;
  };
  
  createdAt: Date;
  updatedAt: Date;
}
```

### 3. Module Gestion Financière

#### 3.1 Facturation
```typescript
interface Invoice {
  id: string;
  invoiceNumber: string; // Auto-généré, unique
  gymId: string;
  memberId: string;
  
  // Dates
  issueDate: Date;
  dueDate: Date;
  paidDate?: Date;
  
  // Montants
  subtotal: number;
  taxRate: number;
  taxAmount: number;
  discount?: {
    type: 'percentage' | 'fixed';
    value: number;
    reason: string;
  };
  total: number;
  currency: string;
  
  // Lignes de facturation
  lineItems: Array<{
    id: string;
    description: string;
    quantity: number;
    unitPrice: number;
    amount: number;
    taxRate: number;
    category: 'subscription' | 'class' | 'personal_training' | 'product' | 'service' | 'late_fee' | 'other';
  }>;
  
  // Statut
  status: 'draft' | 'sent' | 'paid' | 'partial' | 'overdue' | 'cancelled' | 'refunded';
  
  // Paiements associés
  payments: Array<{
    id: string;
    date: Date;
    amount: number;
    method: string;
  }>;
  
  // Documents
  pdfUrl?: string;
  
  // Notes
  notes?: string;
  internalNotes?: string;
  
  // Metadata
  createdAt: Date;
  updatedAt: Date;
  createdBy: string;
}

interface Payment {
  id: string;
  gymId: string;
  memberId: string;
  invoiceId?: string;
  
  // Montant
  amount: number;
  currency: string;
  
  // Méthode de paiement
  paymentMethod: {
    type: 'card' | 'sepa' | 'cash' | 'check' | 'bank_transfer' | 'wallet';
    provider?: 'stripe' | 'paypal' | 'manual';
    last4?: string;
    brand?: string;
  };
  
  // IDs externes
  stripePaymentIntentId?: string;
  stripeChargeId?: string;
  transactionReference?: string;
  
  // Statut
  status: 'pending' | 'processing' | 'succeeded' | 'failed' | 'refunded' | 'disputed';
  failureReason?: string;
  
  // Dates
  paymentDate: Date;
  processedAt?: Date;
  
  // Remboursements
  refunds?: Array<{
    id: string;
    amount: number;
    reason: string;
    date: Date;
  }>;
  
  // Notes
  description?: string;
  notes?: string;
  
  createdAt: Date;
  updatedAt: Date;
}
```

#### 3.2 Gestion des Impayés
```typescript
interface DunningProcess {
  id: string;
  memberId: string;
  invoiceId: string;
  
  // Configuration
  dunningLevel: number; // 1, 2, 3 (escalation)
  
  // Actions automatiques
  actions: Array<{
    level: number;
    daysAfterDue: number;
    action: 'email' | 'sms' | 'suspend_access' | 'cancel_subscription' | 'collection_agency';
    executedAt?: Date;
    status: 'pending' | 'executed' | 'skipped';
  }>;
  
  // Historique communications
  communications: Array<{
    date: Date;
    type: 'email' | 'sms' | 'call' | 'letter';
    template: string;
    sentBy?: string;
    notes?: string;
  }>;
  
  // Résolution
  resolved: boolean;
  resolvedAt?: Date;
  resolutionMethod?: 'payment' | 'payment_plan' | 'write_off' | 'collection';
  
  createdAt: Date;
  updatedAt: Date;
}

interface PaymentPlan {
  id: string;
  memberId: string;
  totalAmount: number;
  
  // Plan
  installments: Array<{
    number: number;
    amount: number;
    dueDate: Date;
    paidAt?: Date;
    paymentId?: string;
    status: 'pending' | 'paid' | 'overdue';
  }>;
  
  // Status
  status: 'active' | 'completed' | 'defaulted' | 'cancelled';
  
  // Terms
  setupFee?: number;
  interestRate?: number;
  
  createdAt: Date;
  updatedAt: Date;
}
```

### 4. Module Communication Marketing

#### 4.1 Campagnes Email
```typescript
interface EmailCampaign {
  id: string;
  gymId: string;
  
  // Identification
  name: string;
  subject: string;
  preheader?: string;
  
  // Contenu
  content: {
    html: string;
    text: string; // version texte
    templateId?: string;
  };
  
  // Segmentation
  segment: {
    type: 'all' | 'custom' | 'segment';
    criteria?: {
      membershipType?: string[];
      status?: string[];
      tags?: string[];
      lastVisit?: {
        operator: 'before' | 'after' | 'between';
        date: Date | [Date, Date];
      };
      customQuery?: any;
    };
    recipientCount: number;
  };
  
  // A/B Testing
  abTest?: {
    enabled: boolean;
    variants: Array<{
      name: string;
      subject: string;
      content: string;
      percentage: number;
    }>;
    winnerMetric: 'open_rate' | 'click_rate' | 'conversion_rate';
  };
  
  // Planification
  scheduling: {
    type: 'immediate' | 'scheduled' | 'recurring';
    sendAt?: Date;
    timezone?: string;
    recurrence?: {
      frequency: 'daily' | 'weekly' | 'monthly';
      interval: number;
      endDate?: Date;
    };
  };
  
  // Statut
  status: 'draft' | 'scheduled' | 'sending' | 'sent' | 'paused' | 'cancelled';
  
  // Statistiques
  stats?: {
    sent: number;
    delivered: number;
    opened: number;
    clicked: number;
    bounced: number;
    unsubscribed: number;
    complained: number;
    
    openRate: number;
    clickRate: number;
    bounceRate: number;
  };
  
  // Tracking
  trackOpens: boolean;
  trackClicks: boolean;
  
  createdAt: Date;
  updatedAt: Date;
  sentAt?: Date;
}
```

#### 4.2 Programme de Parrainage
```typescript
interface ReferralProgram {
  id: string;
  gymId: string;
  
  // Configuration
  name: string;
  description: string;
  isActive: boolean;
  
  // Récompenses
  referrerReward: {
    type: 'discount' | 'credit' | 'free_month' | 'cash' | 'points';
    value: number;
    description: string;
  };
  
  refereeReward: {
    type: 'discount' | 'credit' | 'free_month' | 'cash' | 'points';
    value: number;
    description: string;
  };
  
  // Conditions
  conditions: {
    minSubscriptionMonths?: number; // Durée min pour récompense
    subscriptionTypes?: string[]; // Types éligibles
    maxReferrals?: number; // Max par membre
    validityDays?: number; // Validité du code
  };
  
  // Période
  startDate: Date;
  endDate?: Date;
  
  createdAt: Date;
  updatedAt: Date;
}

interface Referral {
  id: string;
  programId: string;
  
  // Parrain
  referrerId: string;
  referrerCode: string;
  
  // Filleul
  refereeEmail?: string;
  refereeName?: string;
  refereeMemberId?: string; // Une fois inscrit
  
  // Statut
  status: 'pending' | 'signed_up' | 'qualified' | 'rewarded' | 'expired';
  
  // Dates clés
  referredAt: Date;
  signedUpAt?: Date;
  qualifiedAt?: Date;
  rewardedAt?: Date;
  expiresAt?: Date;
  
  // Récompenses distribuées
  referrerRewardGiven: boolean;
  refereeRewardGiven: boolean;
  
  // Tracking
  source?: string; // email, social, direct
  campaignId?: string;
  
  createdAt: Date;
  updatedAt: Date;
}
```

### 5. Module Analytics et Reporting

#### 5.1 KPIs en Temps Réel
```typescript
interface DashboardMetrics {
  gymId: string;
  period: {
    start: Date;
    end: Date;
  };
  
  // Membres
  members: {
    total: number;
    active: number;
    inactive: number;
    new: number;
    churned: number;
    
    byStatus: Record<string, number>;
    bySubscriptionType: Record<string, number>;
    
    retentionRate: number;
    churnRate: number;
  };
  
  // Fréquentation
  attendance: {
    totalVisits: number;
    uniqueMembers: number;
    avgVisitsPerMember: number;
    peakHours: Array<{ hour: number; visits: number }>;
    peakDays: Array<{ day: string; visits: number }>;
    
    byFacility: Record<string, number>;
  };
  
  // Financier
  financial: {
    revenue: {
      total: number;
      subscriptions: number;
      classes: number;
      personal: number;
      products: number;
      other: number;
    };
    
    mrr: number; // Monthly Recurring Revenue
    arr: number; // Annual Recurring Revenue
    arpu: number; // Average Revenue Per User
    
    outstanding: number; // Impayés
    collected: number;
    
    growth: {
      mom: number; // Month over Month
      yoy: number; // Year over Year
    };
  };
  
  // Classes
  classes: {
    totalSessions: number;
    totalAttendees: number;
    avgAttendanceRate: number;
    
    byType: Record<string, {
      sessions: number;
      attendees: number;
      occupancy: number;
    }>;
    
    topClasses: Array<{
      name: string;
      attendees: number;
      occupancy: number;
    }>;
  };
  
  // Performance
  performance: {
    memberSatisfaction: number; // NPS
    staffUtilization: number;
    facilityOccupancy: number;
    
    ltv: number; // Lifetime Value
    cac: number; // Customer Acquisition Cost
    ltvCacRatio: number;
  };
  
  generatedAt: Date;
}
```

#### 5.2 Rapports Personnalisés
```typescript
interface CustomReport {
  id: string;
  gymId: string;
  
  // Configuration
  name: string;
  description?: string;
  type: 'table' | 'chart' | 'dashboard';
  
  // Source de données
  dataSource: {
    entity: 'members' | 'subscriptions' | 'bookings' | 'payments' | 'visits';
    filters: Array<{
      field: string;
      operator: 'equals' | 'not_equals' | 'contains' | 'greater_than' | 'less_than' | 'between';
      value: any;
    }>;
    dateRange: {
      type: 'last_7_days' | 'last_30_days' | 'last_90_days' | 'this_month' | 'last_month' | 'this_year' | 'custom';
      start?: Date;
      end?: Date;
    };
  };
  
  // Métriques et dimensions
  metrics: string[]; // Champs à mesurer
  dimensions: string[]; // Grouper par
  
  // Visualisation
  visualization: {
    type: 'line' | 'bar' | 'pie' | 'doughnut' | 'area' | 'table';
    config: any; // Configuration spécifique au type
  };
  
  // Planification
  schedule?: {
    frequency: 'daily' | 'weekly' | 'monthly';
    day?: number; // Jour du mois ou de la semaine
    time: string; // HH:mm
    recipients: string[]; // Emails
  };
  
  // Métadonnées
  isPublic: boolean;
  createdBy: string;
  lastRun?: Date;
  
  createdAt: Date;
  updatedAt: Date;
}
```

---

## API Endpoints

### Authentication & Authorization

```
POST   /api/v1/auth/register          # Inscription gérant
POST   /api/v1/auth/login             # Connexion
POST   /api/v1/auth/logout            # Déconnexion
POST   /api/v1/auth/refresh           # Refresh token
POST   /api/v1/auth/forgot-password   # Mot de passe oublié
POST   /api/v1/auth/reset-password    # Reset mot de passe
POST   /api/v1/auth/verify-email      # Vérification email
POST   /api/v1/auth/2fa/enable        # Activer 2FA
POST   /api/v1/auth/2fa/verify        # Vérifier code 2FA
```

### Members Management

```
GET    /api/v1/members                # Liste membres (pagination, filtres)
POST   /api/v1/members                # Créer membre
GET    /api/v1/members/:id            # Détail membre
PATCH  /api/v1/members/:id            # Modifier membre
DELETE /api/v1/members/:id            # Supprimer membre (RGPD)
GET    /api/v1/members/:id/history    # Historique membre
POST   /api/v1/members/:id/suspend    # Suspendre membre
POST   /api/v1/members/:id/activate   # Activer membre
GET    /api/v1/members/export         # Export données (CSV, Excel)
POST   /api/v1/members/import         # Import membres
GET    /api/v1/members/search         # Recherche avancée
```

### Subscriptions

```
GET    /api/v1/subscriptions                    # Liste abonnements
POST   /api/v1/subscriptions                    # Créer abonnement
GET    /api/v1/subscriptions/:id                # Détail abonnement
PATCH  /api/v1/subscriptions/:id                # Modifier abonnement
DELETE /api/v1/subscriptions/:id                # Annuler abonnement
POST   /api/v1/subscriptions/:id/suspend        # Suspendre abonnement
POST   /api/v1/subscriptions/:id/resume         # Reprendre abonnement
POST   /api/v1/subscriptions/:id/renew          # Renouveler abonnement

GET    /api/v1/subscription-types               # Types d'abonnement
POST   /api/v1/subscription-types               # Créer type
PATCH  /api/v1/subscription-types/:id           # Modifier type
DELETE /api/v1/subscription-types/:id           # Supprimer type
```

### Access Control

```
GET    /api/v1/access/logs                      # Logs d'accès
POST   /api/v1/access/validate                  # Valider accès
GET    /api/v1/access/stats                     # Statistiques fréquentation
GET    /api/v1/access/devices                   # Liste appareils
POST   /api/v1/access/devices                   # Ajouter appareil
PATCH  /api/v1/access/devices/:id               # Modifier appareil
```

### Scheduling

```
GET    /api/v1/classes                          # Liste cours
POST   /api/v1/classes                          # Créer cours
GET    /api/v1/classes/:id                      # Détail cours
PATCH  /api/v1/classes/:id                      # Modifier cours
DELETE /api/v1/classes/:id                      # Supprimer cours
POST   /api/v1/classes/:id/cancel               # Annuler cours

GET    /api/v1/classes/:id/instances            # Instances du cours
POST   /api/v1/classes/:id/instances            # Créer instance
PATCH  /api/v1/classes/instances/:id            # Modifier instance

GET    /api/v1/bookings                         # Liste réservations
POST   /api/v1/bookings                         # Créer réservation
DELETE /api/v1/bookings/:id                     # Annuler réservation
POST   /api/v1/bookings/:id/checkin             # Marquer présent
POST   /api/v1/bookings/:id/no-show             # Marquer absent

GET    /api/v1/personal-sessions                # Sessions personnelles
POST   /api/v1/personal-sessions                # Créer session
PATCH  /api/v1/personal-sessions/:id            # Modifier session
DELETE /api/v1/personal-sessions/:id            # Annuler session
```

### Payments & Billing

```
GET    /api/v1/invoices                         # Liste factures
POST   /api/v1/invoices                         # Créer facture
GET    /api/v1/invoices/:id                     # Détail facture
PATCH  /api/v1/invoices/:id                     # Modifier facture
POST   /api/v1/invoices/:id/send                # Envoyer facture
POST   /api/v1/invoices/:id/void                # Annuler facture
GET    /api/v1/invoices/:id/pdf                 # Télécharger PDF

GET    /api/v1/payments                         # Liste paiements
POST   /api/v1/payments                         # Créer paiement
GET    /api/v1/payments/:id                     # Détail paiement
POST   /api/v1/payments/:id/refund              # Rembourser

POST   /api/v1/payments/stripe/setup-intent     # Setup Stripe
POST   /api/v1/payments/stripe/payment-intent   # Payment Stripe
POST   /api/v1/payments/webhooks/stripe         # Webhook Stripe

GET    /api/v1/dunning                          # Processus relances
GET    /api/v1/payment-plans                    # Plans de paiement
POST   /api/v1/payment-plans                    # Créer plan
```

### Communication

```
GET    /api/v1/campaigns                        # Campagnes
POST   /api/v1/campaigns                        # Créer campagne
GET    /api/v1/campaigns/:id                    # Détail campagne
PATCH  /api/v1/campaigns/:id                    # Modifier campagne
POST   /api/v1/campaigns/:id/send               # Envoyer campagne
POST   /api/v1/campaigns/:id/pause              # Pause campagne
GET    /api/v1/campaigns/:id/stats              # Statistiques

POST   /api/v1/messages/email                   # Envoyer email individuel
POST   /api/v1/messages/sms                     # Envoyer SMS
POST   /api/v1/messages/push                    # Envoyer push notification

GET    /api/v1/templates                        # Templates communication
POST   /api/v1/templates                        # Créer template
```

### Staff Management

```
GET    /api/v1/staff                            # Liste employés
POST   /api/v1/staff                            # Ajouter employé
GET    /api/v1/staff/:id                        # Détail employé
PATCH  /api/v1/staff/:id                        # Modifier employé
DELETE /api/v1/staff/:id                        # Supprimer employé

GET    /api/v1/staff/:id/schedule               # Planning employé
POST   /api/v1/staff/:id/schedule               # Créer créneau
GET    /api/v1/staff/:id/performance            # Performance employé
```

### Analytics & Reporting

```
GET    /api/v1/analytics/dashboard              # Dashboard principal
GET    /api/v1/analytics/members                # Analytics membres
GET    /api/v1/analytics/financial              # Analytics financier
GET    /api/v1/analytics/classes                # Analytics cours
GET    /api/v1/analytics/attendance             # Analytics fréquentation

GET    /api/v1/reports                          # Rapports personnalisés
POST   /api/v1/reports                          # Créer rapport
GET    /api/v1/reports/:id                      # Exécuter rapport
POST   /api/v1/reports/:id/export               # Export rapport
```

### Facilities & Equipment

```
GET    /api/v1/facilities                       # Espaces/salles
POST   /api/v1/facilities                       # Ajouter espace
PATCH  /api/v1/facilities/:id                   # Modifier espace

GET    /api/v1/equipment                        # Équipements
POST   /api/v1/equipment                        # Ajouter équipement
PATCH  /api/v1/equipment/:id                    # Modifier équipement
POST   /api/v1/equipment/:id/maintenance        # Programmer maintenance
```

### Multi-site

```
GET    /api/v1/gyms                             # Liste salles
POST   /api/v1/gyms                             # Ajouter salle
GET    /api/v1/gyms/:id                         # Détail salle
PATCH  /api/v1/gyms/:id                         # Modifier salle

GET    /api/v1/multi-site/analytics             # Analytics consolidées
GET    /api/v1/multi-site/members/:id           # Membre multi-sites
POST   /api/v1/multi-site/transfer              # Transférer membre
```

### Settings & Configuration

```
GET    /api/v1/settings                         # Configuration générale
PATCH  /api/v1/settings                         # Modifier configuration

GET    /api/v1/integrations                     # Intégrations actives
POST   /api/v1/integrations/:name/connect       # Connecter intégration
DELETE /api/v1/integrations/:name               # Déconnecter

GET    /api/v1/webhooks                         # Webhooks configurés
POST   /api/v1/webhooks                         # Créer webhook
DELETE /api/v1/webhooks/:id                     # Supprimer webhook
```

---

## Sécurité et Conformité

### Headers de Sécurité

```
Content-Security-Policy: default-src 'self'; script-src 'self' 'unsafe-inline'; style-src 'self' 'unsafe-inline'
X-Frame-Options: DENY
X-Content-Type-Options: nosniff
Strict-Transport-Security: max-age=31536000; includeSubDomains
X-XSS-Protection: 1; mode=block
Referrer-Policy: strict-origin-when-cross-origin
Permissions-Policy: geolocation=(), microphone=(), camera=()
```

### Rate Limiting

```
Public API:
├── Authentication: 5 requests/minute/IP
├── Public endpoints: 100 requests/minute/IP
└── Search: 30 requests/minute/user

Authenticated API:
├── GET requests: 1000 requests/hour/user
├── POST/PATCH/DELETE: 200 requests/hour/user
└── Webhooks: 100 requests/minute/endpoint
```

### Encryption

```
Data at Rest:
├── AES-256-GCM pour données sensibles
├── Argon2id pour mots de passe
├── RSA-4096 pour clés
└── Chiffrement transparent BDD (TDE)

Data in Transit:
├── TLS 1.3 minimum
├── Certificate pinning (mobile)
└── Perfect Forward Secrecy
```

---

## Performance et Scalabilité

### Caching Strategy

```
Niveaux de Cache:
├── CDN (Cloudflare): Assets statiques, 1 an
├── Redis: Données fréquemment accédées, 1-24h
│   ├── Session utilisateur: 24h
│   ├── Dashboard metrics: 5min
│   ├── Member profile: 1h
│   └── Class schedules: 15min
├── Application: Query results, 1-5min
└── Database: Query cache, automatic
```

### Database Optimization

```
Indexes:
├── Primary keys: B-tree (automatic)
├── Foreign keys: B-tree
├── Search fields: GIN (full-text)
├── Timestamps: B-tree
└── Composite: Based on query patterns

Partitioning:
├── Access logs: Par mois
├── Payments: Par trimestre
└── Analytics: Par année
```

### Monitoring

```
Métriques Clés:
├── Response time: p50, p95, p99
├── Error rate: < 0.1%
├── Uptime: > 99.9%
├── Database: Query time, connections, deadlocks
├── Cache: Hit rate > 80%
├── Queue: Job processing time, failed jobs
└── Infrastructure: CPU, Memory, Disk, Network
```

---

## Conclusion

Cette spécification technique détaille l'implémentation complète de GymManager Pro, l'application destinée aux gérants de salles de sport. L'architecture est conçue pour être scalable, sécurisée et conforme aux réglementations (RGPD, PCI-DSS, WCAG 2.1 AA, HDS/HIPAA).

### Prochaines Étapes de Développement

1. **Phase 1 - MVP (Mois 1-3)**
   - Authentification et gestion utilisateurs
   - CRUD membres basique
   - Abonnements et facturation simple
   - Planning et réservations

2. **Phase 2 - Core Features (Mois 4-6)**
   - Paiements Stripe
   - Communication (email, SMS)
   - Analytics de base
   - Contrôle d'accès

3. **Phase 3 - Advanced (Mois 7-9)**
   - IA et ML
   - Multi-sites
   - Intégrations tierces
   - Conformité complète

4. **Phase 4 - Scale (Mois 10-12)**
   - Optimisations performance
   - Features premium
   - Marketplace
   - White-label

---

**Version**: 1.0  
**Dernière mise à jour**: 2025-10-21  
**Statut**: Specification complète

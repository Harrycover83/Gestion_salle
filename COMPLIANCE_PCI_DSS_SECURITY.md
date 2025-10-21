# Conformité PCI-DSS et Sécurité - GymManager Pro

## Introduction

Ce document détaille la conformité PCI-DSS (Payment Card Industry Data Security Standard) et les mesures de sécurité pour GymManager Pro. La conformité PCI-DSS est obligatoire pour toute organisation qui traite, stocke ou transmet des données de cartes bancaires.

---

## Table des Matières

1. [Niveau de Conformité](#niveau-de-conformité)
2. [Architecture de Paiement](#architecture-de-paiement)
3. [Les 12 Exigences PCI-DSS](#les-12-exigences-pci-dss)
4. [Tokenisation via Stripe](#tokenisation-via-stripe)
5. [Sécurité Applicative](#sécurité-applicative)
6. [Monitoring et Logging](#monitoring-et-logging)
7. [Tests et Audits](#tests-et-audits)
8. [Incident Response](#incident-response)
9. [Checklist de Conformité](#checklist-de-conformité)

---

## Niveau de Conformité

### SAQ-A (Self-Assessment Questionnaire A)

GymManager Pro vise la conformité **SAQ-A**, le niveau le plus simple, car:
- ✅ Aucun stockage, traitement ou transmission de données de cartes sur nos serveurs
- ✅ Redirection complète vers Stripe (PCI Level 1 Service Provider)
- ✅ Utilisation de Stripe Checkout ou Elements (iFrame isolé)
- ✅ Site HTTPS avec certificat TLS valide

### Critères SAQ-A

```
Éligibilité SAQ-A:
├── Redirection complète vers page de paiement tierce (Stripe Checkout)
│   └── OU: iFrame isolé (Stripe Elements) sans accès aux données cartes
├── Pas de stockage de données cartes
├── Pas de traitement de données cartes
├── Pas de transmission de données cartes via nos systèmes
├── Site web HTTPS uniquement
└── Conformité aux exigences de sécurité web applicables
```

**Nombre d'exigences:** 22 questions (vs 329 pour SAQ-D)  
**Validation annuelle:** Auto-évaluation + Attestation de Conformité (AOC)

---

## Architecture de Paiement

### Flow de Paiement Sécurisé

```
Méthode 1: Stripe Checkout (Recommandée - Plus Simple)
┌─────────────┐
│   Membre    │
└──────┬──────┘
       │ 1. Initie paiement
       ▼
┌─────────────────────┐
│  GymManager Pro     │
│  (Backend)          │
└──────┬──────────────┘
       │ 2. Crée Checkout Session
       │    POST /v1/checkout/sessions
       ▼
┌─────────────────────┐
│   Stripe API        │
└──────┬──────────────┘
       │ 3. Retourne session URL
       ▼
┌─────────────────────┐
│  GymManager Pro     │
│  (Frontend)         │
└──────┬──────────────┘
       │ 4. Redirige vers Stripe
       ▼
┌─────────────────────┐
│  Stripe Checkout    │ ← Hébergé par Stripe, PCI Level 1
│  (Page externe)     │
└──────┬──────────────┘
       │ 5. Membre entre données carte
       │ 6. Stripe traite paiement
       │ 7. Redirige vers success_url
       ▼
┌─────────────────────┐
│  GymManager Pro     │
│  (Confirmation)     │
└──────┬──────────────┘
       │ 8. Webhook reçu
       │    payment_intent.succeeded
       ▼
┌─────────────────────┐
│   Backend           │ ← Valide et enregistre
│   Webhook Handler   │
└─────────────────────┘

Méthode 2: Stripe Elements (Plus de contrôle UX)
┌─────────────────────┐
│  Page de Paiement   │
│  (Notre site)       │
├─────────────────────┤
│                     │
│  [Nom]  [_______]  │ ← Formulaire standard
│                     │
│  ┌─────────────────┐│
│  │  Card Number    ││ ← iFrame Stripe Elements
│  │  [____________] ││    (domaine stripe.com)
│  │                 ││
│  │  Exp   CVV      ││
│  │  [__]  [___]    ││
│  └─────────────────┘│
│                     │
│  [Payer 49.99€]    │
└─────────────────────┘

Nos serveurs ne voient JAMAIS les données de carte
```

### Implémentation Backend

```typescript
// payment.service.ts
import Stripe from 'stripe';

export class PaymentService {
  private stripe: Stripe;
  
  constructor() {
    this.stripe = new Stripe(process.env.STRIPE_SECRET_KEY!, {
      apiVersion: '2023-10-16',
      typescript: true,
    });
  }
  
  /**
   * Méthode 1: Stripe Checkout (Recommandée)
   */
  async createCheckoutSession(
    memberId: string,
    subscriptionTypeId: string,
    successUrl: string,
    cancelUrl: string
  ): Promise<{ sessionId: string; url: string }> {
    const member = await this.memberRepository.findById(memberId);
    const subscriptionType = await this.subscriptionTypeRepository.findById(subscriptionTypeId);
    
    // Créer la session Checkout
    const session = await this.stripe.checkout.sessions.create({
      customer_email: member.email,
      client_reference_id: memberId,
      mode: 'subscription', // ou 'payment' pour one-time
      
      line_items: [
        {
          price_data: {
            currency: 'eur',
            unit_amount: subscriptionType.pricing.basePrice * 100, // centimes
            recurring: {
              interval: subscriptionType.pricing.billingPeriod,
            },
            product_data: {
              name: subscriptionType.name,
              description: subscriptionType.description,
              images: [subscriptionType.imageUrl],
            },
          },
          quantity: 1,
        },
      ],
      
      // Métadonnées pour tracking
      metadata: {
        memberId,
        subscriptionTypeId,
        gymId: member.gymId,
      },
      
      // Webhooks automatiques
      payment_intent_data: {
        metadata: {
          memberId,
          subscriptionTypeId,
        },
      },
      
      // URLs de redirection
      success_url: `${successUrl}?session_id={CHECKOUT_SESSION_ID}`,
      cancel_url: cancelUrl,
      
      // Options
      allow_promotion_codes: true,
      billing_address_collection: 'required',
      
      // Paiements récurrents
      subscription_data: {
        metadata: {
          memberId,
          subscriptionTypeId,
        },
      },
    });
    
    // Logger la création (pas les données de carte!)
    await this.auditLog.log({
      event: 'checkout_session_created',
      memberId,
      sessionId: session.id,
      amount: subscriptionType.pricing.basePrice,
      timestamp: new Date(),
    });
    
    return {
      sessionId: session.id,
      url: session.url!,
    };
  }
  
  /**
   * Méthode 2: Stripe Elements (Plus de contrôle)
   */
  async createPaymentIntent(
    memberId: string,
    amount: number,
    currency: string = 'eur'
  ): Promise<{ clientSecret: string }> {
    const member = await this.memberRepository.findById(memberId);
    
    // Créer PaymentIntent
    const paymentIntent = await this.stripe.paymentIntents.create({
      amount: amount * 100, // centimes
      currency,
      customer: member.stripeCustomerId, // Si déjà créé
      
      metadata: {
        memberId,
        gymId: member.gymId,
      },
      
      // Méthodes de paiement acceptées
      payment_method_types: ['card', 'sepa_debit'],
      
      // Options anti-fraude
      setup_future_usage: 'off_session', // Pour paiements récurrents
    });
    
    return {
      clientSecret: paymentIntent.client_secret!,
    };
  }
  
  /**
   * Webhook Handler - Événements Stripe
   */
  async handleWebhook(
    payload: string | Buffer,
    signature: string
  ): Promise<void> {
    let event: Stripe.Event;
    
    try {
      // Vérifier la signature (sécurité!)
      event = this.stripe.webhooks.constructEvent(
        payload,
        signature,
        process.env.STRIPE_WEBHOOK_SECRET!
      );
    } catch (err) {
      throw new Error(`Webhook signature verification failed: ${err.message}`);
    }
    
    // Traiter l'événement
    switch (event.type) {
      case 'checkout.session.completed':
        await this.handleCheckoutCompleted(event.data.object as Stripe.Checkout.Session);
        break;
        
      case 'payment_intent.succeeded':
        await this.handlePaymentSucceeded(event.data.object as Stripe.PaymentIntent);
        break;
        
      case 'payment_intent.payment_failed':
        await this.handlePaymentFailed(event.data.object as Stripe.PaymentIntent);
        break;
        
      case 'customer.subscription.created':
        await this.handleSubscriptionCreated(event.data.object as Stripe.Subscription);
        break;
        
      case 'customer.subscription.updated':
        await this.handleSubscriptionUpdated(event.data.object as Stripe.Subscription);
        break;
        
      case 'customer.subscription.deleted':
        await this.handleSubscriptionCancelled(event.data.object as Stripe.Subscription);
        break;
        
      case 'invoice.payment_succeeded':
        await this.handleInvoicePaymentSucceeded(event.data.object as Stripe.Invoice);
        break;
        
      case 'invoice.payment_failed':
        await this.handleInvoicePaymentFailed(event.data.object as Stripe.Invoice);
        break;
        
      default:
        console.log(`Unhandled event type: ${event.type}`);
    }
  }
  
  private async handlePaymentSucceeded(paymentIntent: Stripe.PaymentIntent): Promise<void> {
    const memberId = paymentIntent.metadata.memberId;
    
    // Créer le paiement dans notre système
    await this.paymentRepository.create({
      memberId,
      amount: paymentIntent.amount / 100,
      currency: paymentIntent.currency,
      status: 'succeeded',
      stripePaymentIntentId: paymentIntent.id,
      paymentMethod: {
        type: 'card',
        // Stripe nous donne uniquement les 4 derniers chiffres
        last4: paymentIntent.charges.data[0]?.payment_method_details?.card?.last4,
        brand: paymentIntent.charges.data[0]?.payment_method_details?.card?.brand,
      },
      processedAt: new Date(),
    });
    
    // Activer l'abonnement
    if (paymentIntent.metadata.subscriptionTypeId) {
      await this.subscriptionService.activate(
        memberId,
        paymentIntent.metadata.subscriptionTypeId
      );
    }
    
    // Envoyer confirmation
    await this.communicationService.sendPaymentConfirmation(memberId, paymentIntent.amount / 100);
  }
}
```

### Implémentation Frontend

```typescript
// PaymentForm.tsx (avec Stripe Elements)
import { loadStripe } from '@stripe/stripe-js';
import {
  Elements,
  CardElement,
  useStripe,
  useElements,
} from '@stripe/react-stripe-js';

const stripePromise = loadStripe(process.env.REACT_APP_STRIPE_PUBLISHABLE_KEY!);

function CheckoutForm({ amount, memberId }: CheckoutFormProps) {
  const stripe = useStripe();
  const elements = useElements();
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);
  
  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    
    if (!stripe || !elements) {
      return;
    }
    
    setLoading(true);
    setError(null);
    
    try {
      // 1. Créer PaymentIntent côté serveur
      const { clientSecret } = await api.post('/payments/create-intent', {
        amount,
        memberId,
      });
      
      // 2. Confirmer le paiement avec Stripe.js
      const { error: stripeError, paymentIntent } = await stripe.confirmCardPayment(
        clientSecret,
        {
          payment_method: {
            card: elements.getElement(CardElement)!,
            billing_details: {
              name: billingName,
              email: billingEmail,
            },
          },
        }
      );
      
      if (stripeError) {
        setError(stripeError.message);
      } else if (paymentIntent.status === 'succeeded') {
        // Succès!
        onSuccess(paymentIntent.id);
      }
    } catch (err) {
      setError('Une erreur est survenue');
    } finally {
      setLoading(false);
    }
  };
  
  return (
    <form onSubmit={handleSubmit}>
      {/* Champs normaux (pas sensibles) */}
      <input
        type="text"
        placeholder="Nom"
        value={billingName}
        onChange={(e) => setBillingName(e.target.value)}
      />
      
      {/* Élément Stripe (iFrame isolé) */}
      <CardElement
        options={{
          style: {
            base: {
              fontSize: '16px',
              color: '#424770',
              '::placeholder': {
                color: '#aab7c4',
              },
            },
            invalid: {
              color: '#9e2146',
            },
          },
          hidePostalCode: false,
        }}
      />
      
      {error && <div className="error">{error}</div>}
      
      <button type="submit" disabled={!stripe || loading}>
        {loading ? 'Traitement...' : `Payer ${amount}€`}
      </button>
    </form>
  );
}

export function PaymentPage() {
  return (
    <Elements stripe={stripePromise}>
      <CheckoutForm amount={49.99} memberId="mem_123" />
    </Elements>
  );
}
```

---

## Les 12 Exigences PCI-DSS

### Exigence 1: Installer et Maintenir un Pare-feu

```typescript
// Infrastructure Configuration (AWS Security Groups example)
const securityGroupConfig = {
  // Groupe pour Application Load Balancer
  alb: {
    inbound: [
      { port: 443, source: '0.0.0.0/0', protocol: 'tcp' }, // HTTPS public
      { port: 80, source: '0.0.0.0/0', protocol: 'tcp' },  // HTTP (redirect to HTTPS)
    ],
    outbound: [
      { port: 3000, destination: 'app-servers', protocol: 'tcp' },
    ],
  },
  
  // Groupe pour serveurs d'application
  appServers: {
    inbound: [
      { port: 3000, source: 'alb-security-group', protocol: 'tcp' },
      { port: 22, source: 'vpn-ip-range', protocol: 'tcp' }, // SSH via VPN only
    ],
    outbound: [
      { port: 5432, destination: 'db-security-group', protocol: 'tcp' }, // PostgreSQL
      { port: 6379, destination: 'redis-security-group', protocol: 'tcp' }, // Redis
      { port: 443, destination: '0.0.0.0/0', protocol: 'tcp' }, // HTTPS externe (APIs)
    ],
  },
  
  // Groupe pour base de données
  database: {
    inbound: [
      { port: 5432, source: 'app-servers', protocol: 'tcp' },
    ],
    outbound: [], // Pas de sortie
  },
};

// WAF Rules (Web Application Firewall)
const wafRules = {
  // Bloquer SQL injection
  sqlInjectionRule: {
    action: 'BLOCK',
    priority: 1,
    statement: {
      sqlInjectionMatchStatement: {
        fieldToMatch: { allQueryArguments: {} },
        textTransformations: [{ priority: 0, type: 'URL_DECODE' }],
      },
    },
  },
  
  // Bloquer XSS
  xssRule: {
    action: 'BLOCK',
    priority: 2,
    statement: {
      xssMatchStatement: {
        fieldToMatch: { allQueryArguments: {} },
        textTransformations: [{ priority: 0, type: 'URL_DECODE' }],
      },
    },
  },
  
  // Rate limiting
  rateLimitRule: {
    action: 'BLOCK',
    priority: 3,
    statement: {
      rateBasedStatement: {
        limit: 2000, // requêtes par 5 min
        aggregateKeyType: 'IP',
      },
    },
  },
  
  // Bloquer pays haut risque (optionnel)
  geoBlockingRule: {
    action: 'BLOCK',
    priority: 4,
    statement: {
      geoMatchStatement: {
        countryCodes: ['CN', 'RU', 'KP'], // Exemple
      },
    },
  },
};
```

### Exigence 2: Ne Pas Utiliser de Mots de Passe par Défaut

```typescript
// Politique de sécurité des configurations
class SecureConfigurationService {
  async initializeSystem(): Promise<void> {
    // 1. Générer secrets aléatoires
    const jwtSecret = crypto.randomBytes(64).toString('hex');
    const sessionSecret = crypto.randomBytes(64).toString('hex');
    const encryptionKey = crypto.randomBytes(32); // AES-256
    
    // 2. Stocker dans gestionnaire de secrets
    await this.secretsManager.store({
      'jwt-secret': jwtSecret,
      'session-secret': sessionSecret,
      'encryption-key': encryptionKey.toString('base64'),
      'stripe-secret-key': process.env.STRIPE_SECRET_KEY,
      'stripe-webhook-secret': process.env.STRIPE_WEBHOOK_SECRET,
      'database-password': await this.generateStrongPassword(32),
    });
    
    // 3. Configurer accès base de données
    await this.configureDatabase({
      username: 'gymmanager_app',
      password: await this.secretsManager.get('database-password'),
      host: 'postgres-primary.internal',
      port: 5432,
      database: 'gymmanager_prod',
      ssl: {
        rejectUnauthorized: true,
        ca: fs.readFileSync('rds-ca-bundle.pem'),
      },
    });
    
    // 4. Désactiver comptes par défaut
    await this.disableDefaultAccounts();
    
    // 5. Configurer authentification forte
    await this.enableMultiFactorAuth();
  }
  
  private async generateStrongPassword(length: number): Promise<string> {
    const charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*()_+-=[]{}|;:,.<>?';
    const password = crypto.randomBytes(length)
      .reduce((acc, byte) => acc + charset[byte % charset.length], '');
    
    // Vérifier force
    const strength = zxcvbn(password);
    if (strength.score < 4) {
      return this.generateStrongPassword(length); // Régénérer
    }
    
    return password;
  }
  
  private async disableDefaultAccounts(): Promise<void> {
    // Désactiver compte admin par défaut
    await this.userRepository.delete({ username: 'admin' });
    await this.userRepository.delete({ username: 'root' });
    await this.userRepository.delete({ username: 'test' });
    
    // Force création compte admin unique au premier démarrage
    if (await this.userRepository.count() === 0) {
      throw new Error('No admin account configured. Please run setup wizard.');
    }
  }
}
```

### Exigence 3: Protéger les Données Stockées

```typescript
// Chiffrement des données sensibles
class DataEncryptionService {
  private algorithm = 'aes-256-gcm';
  private keyLength = 32; // 256 bits
  
  async encrypt(plaintext: string): Promise<EncryptedData> {
    // Obtenir la clé depuis le gestionnaire de secrets
    const key = await this.getEncryptionKey();
    
    // Générer IV aléatoire
    const iv = crypto.randomBytes(16);
    
    // Créer cipher
    const cipher = crypto.createCipheriv(this.algorithm, key, iv);
    
    // Chiffrer
    let ciphertext = cipher.update(plaintext, 'utf8', 'hex');
    ciphertext += cipher.final('hex');
    
    // Obtenir auth tag
    const authTag = cipher.getAuthTag();
    
    return {
      ciphertext,
      iv: iv.toString('hex'),
      authTag: authTag.toString('hex'),
      algorithm: this.algorithm,
    };
  }
  
  async decrypt(encrypted: EncryptedData): Promise<string> {
    const key = await this.getEncryptionKey();
    
    // Créer decipher
    const decipher = crypto.createDecipheriv(
      encrypted.algorithm,
      key,
      Buffer.from(encrypted.iv, 'hex')
    );
    
    // Définir auth tag
    decipher.setAuthTag(Buffer.from(encrypted.authTag, 'hex'));
    
    // Déchiffrer
    let plaintext = decipher.update(encrypted.ciphertext, 'hex', 'utf8');
    plaintext += decipher.final('utf8');
    
    return plaintext;
  }
  
  // Chiffrement at-rest avec Prisma middleware
  setupDatabaseEncryption(): void {
    prisma.$use(async (params, next) => {
      // Chiffrer avant écriture
      if (params.action === 'create' || params.action === 'update') {
        if (params.model === 'Member' && params.args.data) {
          // Chiffrer champs sensibles
          const sensitiveFields = ['medicalInfo', 'bankAccount'];
          
          for (const field of sensitiveFields) {
            if (params.args.data[field]) {
              params.args.data[field] = await this.encrypt(
                JSON.stringify(params.args.data[field])
              );
            }
          }
        }
      }
      
      // Exécuter
      const result = await next(params);
      
      // Déchiffrer après lecture
      if (params.action === 'findUnique' || params.action === 'findMany') {
        if (params.model === 'Member') {
          const decrypt = async (member: any) => {
            const sensitiveFields = ['medicalInfo', 'bankAccount'];
            
            for (const field of sensitiveFields) {
              if (member[field]) {
                const decrypted = await this.decrypt(member[field]);
                member[field] = JSON.parse(decrypted);
              }
            }
          };
          
          if (Array.isArray(result)) {
            await Promise.all(result.map(decrypt));
          } else if (result) {
            await decrypt(result);
          }
        }
      }
      
      return result;
    });
  }
  
  // Rotation des clés
  async rotateEncryptionKeys(): Promise<void> {
    // 1. Générer nouvelle clé
    const newKey = crypto.randomBytes(this.keyLength);
    
    // 2. Stocker dans gestionnaire
    await this.secretsManager.store({
      'encryption-key-new': newKey.toString('base64'),
      'encryption-key-old': await this.secretsManager.get('encryption-key'),
    });
    
    // 3. Re-chiffrer toutes les données
    await this.reEncryptAllData(newKey);
    
    // 4. Promouvoir nouvelle clé
    await this.secretsManager.store({
      'encryption-key': newKey.toString('base64'),
    });
    
    // 5. Supprimer ancienne clé (après période de grâce)
    setTimeout(async () => {
      await this.secretsManager.delete('encryption-key-old');
    }, 30 * 24 * 60 * 60 * 1000); // 30 jours
  }
}
```

### Exigence 4: Chiffrer les Transmissions

```typescript
// Configuration TLS/HTTPS
import https from 'https';
import fs from 'fs';

class SecureServerConfig {
  createHTTPSServer(): https.Server {
    const options: https.ServerOptions = {
      // Certificats SSL/TLS
      cert: fs.readFileSync('/etc/ssl/certs/server.crt'),
      key: fs.readFileSync('/etc/ssl/private/server.key'),
      ca: fs.readFileSync('/etc/ssl/certs/ca-bundle.crt'),
      
      // Versions TLS
      minVersion: 'TLSv1.3', // Minimum TLS 1.3
      maxVersion: 'TLSv1.3',
      
      // Ciphers forts uniquement
      ciphers: [
        'TLS_AES_256_GCM_SHA384',
        'TLS_CHACHA20_POLY1305_SHA256',
        'TLS_AES_128_GCM_SHA256',
      ].join(':'),
      
      // Perfect Forward Secrecy
      honorCipherOrder: true,
      
      // OCSP Stapling
      requestOCSP: true,
    };
    
    return https.createServer(options, this.app);
  }
  
  // Headers de sécurité
  configureSecurityHeaders(): void {
    this.app.use((req, res, next) => {
      // HTTPS strict
      res.setHeader(
        'Strict-Transport-Security',
        'max-age=31536000; includeSubDomains; preload'
      );
      
      // Content Security Policy
      res.setHeader(
        'Content-Security-Policy',
        "default-src 'self'; " +
        "script-src 'self' https://js.stripe.com; " +
        "style-src 'self' 'unsafe-inline'; " +
        "frame-src https://js.stripe.com https://hooks.stripe.com; " +
        "connect-src 'self' https://api.stripe.com; " +
        "img-src 'self' data: https:;"
      );
      
      // Autres headers
      res.setHeader('X-Frame-Options', 'DENY');
      res.setHeader('X-Content-Type-Options', 'nosniff');
      res.setHeader('X-XSS-Protection', '1; mode=block');
      res.setHeader('Referrer-Policy', 'strict-origin-when-cross-origin');
      
      // Pas de cache pour données sensibles
      if (req.path.includes('/api/')) {
        res.setHeader('Cache-Control', 'no-store, no-cache, must-revalidate, private');
        res.setHeader('Pragma', 'no-cache');
      }
      
      next();
    });
  }
  
  // Redirection HTTP -> HTTPS
  setupHTTPRedirect(): void {
    const httpApp = express();
    httpApp.all('*', (req, res) => {
      res.redirect(301, `https://${req.hostname}${req.url}`);
    });
    httpApp.listen(80);
  }
}
```

### Exigences 5-12: Résumé des Implémentations

```typescript
// Exigence 5: Protéger contre les Malware
class MalwareProtectionService {
  async scanUploadedFile(file: Express.Multer.File): Promise<ScanResult> {
    // Utiliser ClamAV ou service externe
    const scanner = new ClamScan();
    const result = await scanner.scanFile(file.path);
    
    if (!result.isInfected) {
      return { safe: true };
    } else {
      // Supprimer fichier infecté
      fs.unlinkSync(file.path);
      
      // Logger
      await this.auditLog.log({
        event: 'malware_detected',
        filename: file.originalname,
        virus: result.viruses.join(', '),
        timestamp: new Date(),
      });
      
      throw new MalwareDetectedError('File contains malware');
    }
  }
}

// Exigence 6: Développer des Systèmes Sécurisés
class SecureDevelopmentPractices {
  // SAST - Static Application Security Testing
  async runSecurityScan(): Promise<void> {
    // Dans CI/CD pipeline
    await exec('npm audit --audit-level=moderate');
    await exec('snyk test --severity-threshold=high');
    await exec('sonarqube-scanner');
  }
  
  // Input Validation
  validateInput(data: any, schema: Joi.Schema): any {
    const { error, value } = schema.validate(data, {
      abortEarly: false,
      stripUnknown: true,
    });
    
    if (error) {
      throw new ValidationError(error.details);
    }
    
    return value;
  }
  
  // Parameterized Queries (ORM Prisma)
  async safeQuery(memberId: string): Promise<Member> {
    // ✅ Bon - Requête paramétrée
    return await prisma.member.findUnique({
      where: { id: memberId },
    });
    
    // ❌ Mauvais - SQL injection possible
    // return await prisma.$queryRaw`SELECT * FROM members WHERE id = ${memberId}`;
  }
}

// Exigence 7: Restreindre l'Accès (Need-to-Know)
// Déjà implémenté dans RBAC (voir section précédente)

// Exigence 8: Identifier et Authentifier
// Voir section Authentication plus bas

// Exigence 9: Restreindre l'Accès Physique
// Contrôles physiques du datacenter (géré par cloud provider)

// Exigence 10: Logger et Monitorer
// Voir section Monitoring plus bas

// Exigence 11: Tester la Sécurité
// Voir section Tests plus bas

// Exigence 12: Politique de Sécurité
class SecurityPolicyService {
  async generateSecurityPolicy(): Promise<SecurityPolicy> {
    return {
      version: '1.0',
      effectiveDate: new Date(),
      
      sections: {
        accessControl: {
          passwordPolicy: {
            minLength: 12,
            requireUppercase: true,
            requireLowercase: true,
            requireNumbers: true,
            requireSpecialChars: true,
            maxAge: 90, // jours
            historyCount: 5, // Pas de réutilisation
          },
          
          mfaPolicy: {
            required: true,
            methods: ['totp', 'sms'],
          },
          
          sessionPolicy: {
            timeout: 30, // minutes
            absoluteTimeout: 8, // heures
          },
        },
        
        dataProtection: {
          encryption: {
            atRest: 'AES-256-GCM',
            inTransit: 'TLS 1.3',
            keyRotation: 365, // jours
          },
          
          dataRetention: RetentionPolicies,
        },
        
        incidentResponse: {
          contacts: ['security@gymmanager.com'],
          escalationProcedure: '...',
          reportingDeadline: 72, // heures
        },
        
        vendorManagement: {
          dueDiligence: true,
          dataProcessingAgreements: true,
          regularAudits: true,
        },
        
        training: {
          frequency: 'annual',
          topics: ['GDPR', 'PCI-DSS', 'Phishing', 'Passwords'],
        },
      },
    };
  }
}
```

---

## Monitoring et Logging

### Audit Logs PCI-DSS Compliant

```typescript
interface AuditLog {
  // Qui
  userId: string;
  userEmail: string;
  userRole: string;
  
  // Quoi
  event: string;
  action: 'create' | 'read' | 'update' | 'delete';
  resource: string;
  resourceId?: string;
  
  // Quand
  timestamp: Date;
  
  // Où
  ipAddress: string;
  userAgent: string;
  geolocation?: {
    country: string;
    city: string;
  };
  
  // Comment
  method: string; // HTTP method
  endpoint: string;
  statusCode: number;
  
  // Résultat
  success: boolean;
  errorMessage?: string;
  
  // Données
  requestData?: any; // Sanitized - no sensitive data!
  responseData?: any; // Sanitized
  
  // Contexte
  sessionId: string;
  requestId: string;
  
  // Sécurité
  anomalyScore?: number;
  riskLevel?: 'low' | 'medium' | 'high';
}

class AuditLogService {
  async log(entry: Partial<AuditLog>): Promise<void> {
    // Enrichir avec contexte
    const enriched: AuditLog = {
      ...entry,
      timestamp: new Date(),
      requestId: AsyncLocalStorage.getStore()?.requestId,
      sessionId: AsyncLocalStorage.getStore()?.sessionId,
    } as AuditLog;
    
    // Stocker dans base dédiée (append-only)
    await this.auditLogRepository.create(enriched);
    
    // Envoyer vers SIEM (Security Information and Event Management)
    await this.siemService.send(enriched);
    
    // Détecter anomalies
    if (this.isAnomalous(enriched)) {
      await this.alertingService.alert({
        severity: 'high',
        message: `Anomalous activity detected: ${enriched.event}`,
        details: enriched,
      });
    }
  }
  
  // Événements critiques à logger (PCI-DSS 10.2)
  async logCriticalEvent(event: CriticalEvent): Promise<void> {
    await this.log({
      event: event.type,
      ...event.data,
    });
    
    // Notification immédiate pour événements critiques
    if (event.severity === 'critical') {
      await this.notifySecurityTeam(event);
    }
  }
  
  // Revue régulière des logs (PCI-DSS 10.6)
  async scheduleLogReview(): void {
    // Quotidien: Revue automatique
    cron.schedule('0 1 * * *', async () => {
      const report = await this.generateDailySecurityReport();
      await this.emailService.send({
        to: 'security@gymmanager.com',
        subject: 'Daily Security Report',
        body: report,
      });
    });
    
    // Hebdomadaire: Revue manuelle
    // Mensuel: Revue complète
  }
}

// Événements critiques PCI-DSS 10.2
enum CriticalEventType {
  // 10.2.1 Accès cardholder data
  CARDHOLDER_DATA_ACCESS = 'cardholder_data_access',
  
  // 10.2.2 Actions admin
  ADMIN_ACTION = 'admin_action',
  
  // 10.2.3 Accès audit logs
  AUDIT_LOG_ACCESS = 'audit_log_access',
  
  // 10.2.4 Accès invalide
  INVALID_ACCESS_ATTEMPT = 'invalid_access_attempt',
  
  // 10.2.5 Identification/authentication
  AUTH_EVENT = 'auth_event',
  
  // 10.2.6 Initialisation logs
  AUDIT_LOG_INIT = 'audit_log_init',
  
  // 10.2.7 Création/suppression objets système
  SYSTEM_OBJECT_CHANGE = 'system_object_change',
}
```

---

## Tests et Audits

### Vulnerability Scanning (PCI-DSS 11.2)

```typescript
// Scan trimestriel par ASV (Approved Scanning Vendor)
class VulnerabilityScanning {
  async runQuarterlyScan(): Promise<ScanReport> {
    // Utiliser service externe certifié
    const asv = new ApprovedScanningVendor({
      apiKey: process.env.ASV_API_KEY,
    });
    
    const scan = await asv.startScan({
      target: 'https://app.gymmanager.com',
      type: 'external',
      compliance: 'PCI-DSS',
    });
    
    // Attendre complétion
    const report = await scan.waitForCompletion();
    
    // Vérifier conformité
    if (!report.passed) {
      await this.createRemediationTasks(report.vulnerabilities);
      await this.notifySecurityTeam(report);
    }
    
    // Archiver
    await this.archiveScanReport(report);
    
    return report;
  }
  
  async runInternalScan(): Promise<void> {
    // Scan interne trimestriel
    await exec('nmap -sV --script vuln your-internal-ips');
    await exec('openvas-start --target your-internal-ips');
  }
}
```

### Penetration Testing (PCI-DSS 11.3)

```typescript
// Pentest annuel
class PenetrationTesting {
  async schedulePentest(): Promise<void> {
    // Engager société spécialisée
    const pentest = new PentestProvider({
      scope: [
        'https://app.gymmanager.com',
        'https://api.gymmanager.com',
      ],
      type: 'external',
      methodology: 'OWASP',
      compliance: 'PCI-DSS',
    });
    
    const report = await pentest.execute();
    
    // Plan de remediation
    await this.createRemediationPlan(report.findings);
    
    // Re-test après corrections
    await pentest.retest();
  }
}
```

---

## Incident Response

```typescript
class IncidentResponseService {
  async handleSecurityIncident(incident: SecurityIncident): Promise<void> {
    // 1. Détection et alerte
    await this.alertSecurityTeam(incident);
    
    // 2. Containment
    if (incident.severity === 'critical') {
      await this.isolateAffectedSystems();
    }
    
    // 3. Investigation
    const investigation = await this.investigate(incident);
    
    // 4. Notification (si breach données)
    if (investigation.isDataBreach) {
      await this.notifyAuthorities(investigation); // CNIL dans 72h
      await this.notifyAffectedIndividuals(investigation);
      await this.notifyPaymentBrands(investigation); // Visa, Mastercard
    }
    
    // 5. Remediation
    await this.remediate(investigation.findings);
    
    // 6. Post-mortem
    await this.conductPostMortem(incident, investigation);
    
    // 7. Update policies
    await this.updateSecurityPolicies(investigation.lessons);
  }
}
```

---

## Checklist de Conformité

```markdown
### PCI-DSS SAQ-A Checklist

#### Build and Maintain a Secure Network
- [ ] 1.1 Firewall configuré
- [ ] 1.2 Configurations sécurisées (pas de défauts)
- [ ] 2.1 Mots de passe par défaut changés
- [ ] 2.2 Configurations système sécurisées

#### Protect Cardholder Data
- [ ] 3.1 **Pas de stockage de données cartes** ✅ (Stripe uniquement)
- [ ] 3.4 PAN rendu illisible (N/A - pas de stockage)

#### Maintain a Vulnerability Management Program
- [ ] 5.1 Anti-malware déployé
- [ ] 6.1 Processus de gestion des vulnérabilités
- [ ] 6.2 Patches de sécurité appliqués

#### Implement Strong Access Control
- [ ] 7.1 Accès limité (need-to-know)
- [ ] 8.1 Utilisateurs identifiés de manière unique
- [ ] 8.2 Authentification forte
- [ ] 8.3 MFA pour accès admin
- [ ] 9.1 Contrôles d'accès physiques

#### Regularly Monitor and Test Networks
- [ ] 10.1 Audit trail pour accès données cartes
- [ ] 10.2 Logs d'événements critiques
- [ ] 10.3 Logs protégés
- [ ] 10.4 Revue régulière des logs
- [ ] 11.1 Test de présence de wireless non autorisé
- [ ] 11.2 Scans de vulnérabilités trimestriels (ASV)
- [ ] 11.3 Penetration testing annuel

#### Maintain an Information Security Policy
- [ ] 12.1 Politique de sécurité
- [ ] 12.2 Analyse de risque annuelle
- [ ] 12.3 Politique d'usage acceptable
- [ ] 12.4 Responsabilités de sécurité
- [ ] 12.5 Responsabilités assignées
- [ ] 12.6 Programme de sensibilisation
- [ ] 12.8 Gestion des prestataires

#### SAQ-A Specific
- [ ] ✅ Redirection complète vers Stripe
- [ ] ✅ iFrame Stripe isolé (pas d'accès aux données)
- [ ] ✅ Pas de stockage, traitement ou transmission
- [ ] ✅ Site HTTPS uniquement
- [ ] ✅ TLS 1.2+ configuré

#### Annual Requirements
- [ ] Scan ASV trimestriel (4/an)
- [ ] Pentest annuel
- [ ] Formation sécurité annuelle
- [ ] Revue politique de sécurité
- [ ] Attestation de Conformité (AOC)
```

---

**Version**: 1.0  
**Dernière mise à jour**: 2025-10-21  
**Prochaine revue**: 2026-01-21  
**Statut**: Actif

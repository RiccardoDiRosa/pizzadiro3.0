# Configurazione Pagamenti Sicuri

Questo progetto implementa un sistema di pagamento sicuro utilizzando **Stripe** per le carte di credito/debito e **PayPal** per i pagamenti PayPal.

## 🔒 Sicurezza

Il sistema è progettato seguendo le migliori pratiche di sicurezza:

- ✅ **Nessun dato della carta viene mai salvato sul nostro server**
- ✅ **Tokenizzazione**: Stripe gestisce tutti i dati sensibili direttamente
- ✅ **PCI DSS Compliant**: Stripe è certificato PCI DSS Level 1
- ✅ **Crittografia end-to-end**: Tutti i dati sono crittografati durante la trasmissione
- ✅ **Validazione lato client e server**: I dati vengono validati prima dell'invio

## 📋 Metodi di Pagamento Supportati

1. **Carta di Credito/Debito** (Stripe)
   - Visa, Mastercard, American Express, Discover
   - Supporta carte di debito e credito
   - Validazione in tempo reale

2. **PayPal**
   - Pagamento tramite account PayPal
   - Supporta carte collegate a PayPal

3. **Contrassegno**
   - Pagamento in contanti alla consegna

## 🚀 Setup

### 1. Configurazione Stripe

1. Crea un account su [Stripe](https://stripe.com)
2. Vai su [Dashboard > API Keys](https://dashboard.stripe.com/apikeys)
3. Copia la **Publishable Key** (chiave pubblica)
4. Aggiungila al file `.env`:
   ```
   VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
   ```

**IMPORTANTE**: 
- Usa le chiavi di **TEST** durante lo sviluppo
- Usa le chiavi **LIVE** solo in produzione

### 2. Configurazione PayPal

1. Crea un account su [PayPal Developer](https://developer.paypal.com)
2. Vai su [Dashboard > My Apps & Credentials](https://developer.paypal.com/dashboard/applications)
3. Crea una nuova app (Sandbox per test, Live per produzione)
4. Copia il **Client ID**
5. Aggiungilo al file `.env`:
   ```
   VITE_PAYPAL_CLIENT_ID=...
   ```

**IMPORTANTE**:
- Usa il Client ID di **SANDBOX** durante lo sviluppo
- Usa il Client ID **LIVE** solo in produzione

### 3. Backend per Stripe (Raccomandato)

Per sicurezza, i Payment Intents di Stripe dovrebbero essere creati dal backend usando la **Secret Key** (mai esporre la Secret Key nel frontend!).

#### Opzione A: Supabase Edge Functions (Raccomandato)

Crea una Supabase Edge Function per gestire i Payment Intents:

```typescript
// supabase/functions/create-payment-intent/index.ts
import { serve } from "https://deno.land/std@0.168.0/http/server.ts"
import Stripe from "https://esm.sh/stripe@14.21.0"

const stripe = new Stripe(Deno.env.get('STRIPE_SECRET_KEY') || '', {
  apiVersion: '2023-10-16',
})

serve(async (req) => {
  try {
    const { amount, currency = 'eur' } = await req.json()

    const paymentIntent = await stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Stripe usa centesimi
      currency: currency.toLowerCase(),
    })

    return new Response(
      JSON.stringify({ clientSecret: paymentIntent.client_secret }),
      { headers: { "Content-Type": "application/json" } }
    )
  } catch (error) {
    return new Response(
      JSON.stringify({ error: error.message }),
      { status: 400, headers: { "Content-Type": "application/json" } }
    )
  }
})
```

Poi aggiorna `lib/stripe.ts` per chiamare questa funzione:

```typescript
export const createPaymentIntent = async (amount: number, currency: string = 'EUR') => {
  const { data, error } = await supabase.functions.invoke('create-payment-intent', {
    body: { amount, currency }
  })
  
  if (error) throw error
  return data.clientSecret
}
```

#### Opzione B: Backend Separato

Crea un endpoint API nel tuo backend che crea Payment Intents usando la Stripe Secret Key.

### 4. Variabili d'Ambiente

Crea un file `.env` nella root del progetto (vedi `.env.example` per riferimento):

```env
VITE_STRIPE_PUBLISHABLE_KEY=pk_test_...
VITE_PAYPAL_CLIENT_ID=...
VITE_API_URL=https://tuo-backend.com  # Opzionale, se usi un backend separato
```

## 🧪 Test

### Carte di Test Stripe

Usa queste carte per testare i pagamenti:

- **Successo**: `4242 4242 4242 4242`
- **Richiede autenticazione**: `4000 0025 0000 3155`
- **Rifiutata**: `4000 0000 0000 0002`

Usa qualsiasi CVV futuro (es. 12/34) e qualsiasi CAP.

### Test PayPal

Usa l'account Sandbox PayPal per testare i pagamenti PayPal.

## 📝 Note Importanti

1. **NON salvare mai dati della carta**: I dati della carta vengono gestiti direttamente da Stripe/PayPal
2. **Usa HTTPS in produzione**: I pagamenti richiedono HTTPS per funzionare correttamente
3. **Valida sempre lato server**: Anche se validi lato client, valida sempre lato server
4. **Gestisci gli errori**: Implementa una gestione degli errori robusta
5. **Logging**: Registra i tentativi di pagamento per audit e debugging

## 🔍 Troubleshooting

### Stripe non funziona
- Verifica che `VITE_STRIPE_PUBLISHABLE_KEY` sia configurato correttamente
- Assicurati di usare una chiave di TEST durante lo sviluppo
- Controlla la console del browser per errori

### PayPal non funziona
- Verifica che `VITE_PAYPAL_CLIENT_ID` sia configurato correttamente
- Assicurati di usare un Client ID di SANDBOX durante lo sviluppo
- Controlla la console del browser per errori

### Payment Intent non viene creato
- Se usi un backend, verifica che l'endpoint sia raggiungibile
- Verifica che la Stripe Secret Key sia configurata correttamente nel backend
- Controlla i log del backend per errori

## 📚 Risorse

- [Stripe Documentation](https://stripe.com/docs)
- [PayPal Developer Documentation](https://developer.paypal.com/docs)
- [Stripe Security Best Practices](https://stripe.com/docs/security)
- [PCI DSS Compliance](https://www.pcisecuritystandards.org/)


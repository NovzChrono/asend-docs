# AsendService - Documentation Laravel

## Vue d'ensemble

Le service `AsendService` permet d'envoyer des SMS via l'API Asend. Il supporte l'envoi de messages simples et en masse (bulk).

## Configuration

### Variables d'environnement

Ajoutez ces variables dans votre fichier `.env` :

```env
ASEND_BASE_URL=https://asend-api.aspace.verif.ci/api/v1/
ASEND_API_TOKEN=votre_token_api
ASEND_SENDER_ID=votre_sender_id
```

### Fichier de configuration

Dans `config/services.php`, ajoutez :

```php
'asend' => [
    'base_url' => env('ASEND_BASE_URL'),
    'api_token' => env('ASEND_API_TOKEN'),
    'sender_id' => env('ASEND_SENDER_ID'),
],
```

## DTOs (Data Transfer Objects)

### SingleMessageDto

Utilisé pour envoyer un message simple à un seul destinataire.

```php
namespace App\Dto;

class SingleMessageDto
{
    public function __construct(
        public string $recipient,      // Numéro de téléphone au format international (+2250704051152)
        public string $message,        // Contenu du message
        public string $senderId, // Nom de l'expéditeur 
    ) {}
}
```

### BulkMessageDto

Utilisé pour envoyer des messages en masse à plusieurs destinataires.

```php

<?php

namespace App\Dto;

class BulkMessageDto
{
    public function __construct(
        public array $recipients,      // Tableau de destinataires
        public string $message,        // Message par défaut
        public string|null $sender = null, // Nom de l'expéditeur (optionnel)
    ) {}
}
```

### Exemple de service

```php

<?php

namespace App\Services;

use App\Dto\BulkMessageDto;
use App\Dto\BulkMessageResponseDto;
use App\Dto\SingleMessageDto;
use App\Dto\SingleMessageResponseDto;
use Exception;
use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Log;
use Illuminate\Http\Client\RequestException;

class AsendService
{
    private const header = [
        'Accept'        => 'application/json',
        'Content-Type'  => 'application/json',
    ];
    private array $aggregator;

    public function __construct()
    {
        $this->aggregator = [
            'baseUrl'  => config('services.asend.base_url'),
            'apiToken' => config('services.asend.api_token'),
            'senderId' => config('services.asend.sender_id'),
        ];
    }

    public function singleMessage(SingleMessageDto $data): SingleMessageResponseDto
    {
        try {
            $url = $this->aggregator['baseUrl'] . 'messages/send';

            $response = Http::withHeaders(array_merge(self::header, [
                'x-api-token'   => $this->aggregator['apiToken'],
            ]))
                ->timeout(50)
                ->post($url, [
                    'to'      => $data->recipient,
                    'message' => $data->message,
                    'from'    => $data->sender ?? $this->aggregator['senderId'],
                ]);

            $response->throw();

            return SingleMessageResponseDto::fromArray($response->json());
        } catch (RequestException $e) {
            Log::error(
                message: 'Error in singleMessage: ',
                context: [
                    $e->response?->status(),
                    $e->response?->json('message'),
                ]
            );

            throw new Exception(
                $e->response?->json('message')
                    ?? 'An error occurred while sending the message'
            );
        }
    }

    public function bulkMessage(BulkMessageDto $data): BulkMessageResponseDto
    {
        try {
            $url = $this->aggregator['baseUrl'] . 'messages/send/bulk';

            Log::info($url);

            $response = Http::withHeaders(array_merge(self::header, [
                'x-api-token'   => $this->aggregator['apiToken'],
            ]))
                ->timeout(50)
                ->post($url, [
                    'recipients' => $data->recipients,
                    'message'    => $data->message,
                    'from'       => $data->sender ?? $this->aggregator['senderId'],
                ]);

            $response->throw();

            Log::info($response->json());

            return BulkMessageResponseDto::fromArray($response->json());
        } catch (RequestException $e) {
            Log::error(
                message: 'Error in bulkMessage: ',
                context: [
                    $e->response?->status(),
                    $e->response?->json('message'),
                ]
            );

            throw new Exception(
                $e->response?->json('message')
                    ?? 'An error occurred while sending bulk messages'
            );
        }
    }
}

```

### Exemple de controller

```php

<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Services\AsendService;
use App\Dto\SingleMessageDto;
use Illuminate\Http\Request;
use Illuminate\Http\JsonResponse;

class SmsController extends Controller
{
    public function __construct(
        private AsendService $asendService
    ) {}

    public function send(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'recipient' => 'required|string|regex:/^\+[0-9]{10,15}$/',
            'message' => 'required|string|max:160',
            'sender' => 'nullable|string|max:11',
        ]);

        try {
            $result = $this->asendService->singleMessage(
                new SingleMessageDto(
                    recipient: $validated['recipient'],
                    message: $validated['message'],
                    sender: $validated['sender'] ?? null
                )
            );

            return response()->json([
                'status' => 'success',
                'message' => 'SMS envoyé avec succès',
                'data' => $result
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'status' => 'error',
                'message' => $e->getMessage()
            ], 500);
        }
    }

    public function sendBulk(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'recipients' => 'required|array|min:1',
            'recipients.*.to' => 'required|string|regex:/^\+[0-9]{10,15}$/',
            'recipients.*.message' => 'nullable|string|max:160',
            'message' => 'required|string|max:160',
            'sender' => 'nullable|string|max:11',
        ]);

        try {
            $result = $this->asendService->bulkMessage(
                new BulkMessageDto(
                    recipients: $validated['recipients'],
                    message: $validated['message'],
                    sender: $validated['sender'] ?? null
                )
            );

            return response()->json([
                'status' => 'success',
                'message' => 'SMS en masse envoyés avec succès',
                'data' => $result
            ]);
        } catch (\Exception $e) {
            return response()->json([
                'status' => 'error',
                'message' => $e->getMessage()
            ], 500);
        }
    }
}
```

## Routes de test

Dans `routes/api.php` :

```php
use App\Services\AsendService;
use App\Dto\SingleMessageDto;
use App\Dto\BulkMessageDto;

Route::group(['prefix' => 'asend'], function () {

    // Test message simple
    Route::get('simple', function () {
        $result = (new AsendService())->singleMessage(
            new SingleMessageDto(
                recipient: '+2250704051152',
                message: 'This is a test message from single route',
                sender: 'Asernum',
            )
        );

        return response()->json([
            'status' => 'success',
            'data' => $result
        ]);
    });

    // Test message en masse
    Route::get('bulk', function () {
        $result = (new AsendService())->bulkMessage(
            new BulkMessageDto(
                recipients: [
                    ['to' => '+2250704051152'],
                    [
                        'to' => '+2250565476902',
                        'message' => 'This is a test message from bulk route personalized',
                    ]
                ],
                message: 'This is a test message from bulk route',
                sender: 'Asernum',
            )
        );

        return response()->json([
            'status' => 'success',
            'data' => $result
        ]);
    });
});
```

## Format de réponse

### SingleMessageResponseDto

```json
{
    "id": "c8bf8dc2-ee77-4e34-a9c5-8dcacfc754de",
    "recipientPhone": "+2250704051152",
    "content": "msg_292zeejddd",
    "cost": 15,
    "status": "SUBMITTED",
}
```

### BulkMessageResponseDto

```json
{
    "total": 2,
    "accepted": 2,
    "rejected": 0,
    "totalCost": 30,
    "messages": [
        {
            "message_id": "c8bf8dc2-ee77-4e34-a9c5-8dcacfc754de",
            "recipientPhone": "+2250704051152",
            "status": "SUBMITTED"
        },
        {
            "message_id": "c8bf8dc2-ee77-4e34-a9c5-8dcacfc754ff",
            "recipientPhone": "+2250565476902",
            "status": "SUBMITTED"
        }
    ]
}
```

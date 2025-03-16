c:\Users\saihe\Zai_Codes\Next.js\clothing-store\clothing-store\
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/
│   │   │       └── app/
│   │   │           └── clothing_store/
│   │   │               ├── config/
│   │   │               │   ├── SecurityConfig.java
│   │   │               │   └── WebConfig.java
│   │   │               ├── controller/
│   │   │               │   ├── AuthController.java
│   │   │               │   ├── CategoryController.java
│   │   │               │   ├── OrderController.java
│   │   │               │   ├── ProductController.java
│   │   │               │   └── UserController.java
│   │   │               ├── dto/
│   │   │               │   ├── request/
│   │   │               │   │   ├── LoginRequest.java
│   │   │               │   │   ├── RegisterRequest.java
│   │   │               │   │   ├── ProductRequest.java
│   │   │               │   │   └── OrderRequest.java
│   │   │               │   └── response/
│   │   │               │       ├── AuthResponse.java
│   │   │               │       ├── ProductResponse.java
│   │   │               │       └── ApiResponse.java
│   │   │               ├── exception/
│   │   │               │   ├── GlobalExceptionHandler.java
│   │   │               │   ├── ResourceNotFoundException.java
│   │   │               │   └── BadRequestException.java
│   │   │               ├── model/
│   │   │               │   ├── User.java
│   │   │               │   ├── Address.java
│   │   │               │   ├── Product.java
│   │   │               │   ├── Category.java
│   │   │               │   ├── ProductVariant.java
│   │   │               │   ├── ProductImage.java
│   │   │               │   ├── Order.java
│   │   │               │   └── OrderItem.java
│   │   │               ├── repository/
│   │   │               │   ├── UserRepository.java
│   │   │               │   ├── AddressRepository.java
│   │   │               │   ├── ProductRepository.java
│   │   │               │   ├── CategoryRepository.java
│   │   │               │   ├── ProductVariantRepository.java
│   │   │               │   ├── ProductImageRepository.java
│   │   │               │   ├── OrderRepository.java
│   │   │               │   └── OrderItemRepository.java
│   │   │               ├── service/
│   │   │               │   ├── AuthService.java
│   │   │               │   ├── UserService.java
│   │   │               │   ├── ProductService.java
│   │   │               │   ├── CategoryService.java
│   │   │               │   ├── OrderService.java
│   │   │               │   └── FileStorageService.java
│   │   │               ├── security/
│   │   │               │   ├── JwtTokenProvider.java
│   │   │               │   ├── JwtAuthenticationFilter.java
│   │   │               │   └── UserDetailsServiceImpl.java
│   │   │               └── ClothingStoreApplication.java
│   │   └── resources/
│   │       ├── application.properties
│   │       ├── application-dev.properties
│   │       └── application-prod.properties
│   └── test/
│       └── java/
│           └── com/
│               └── app/
│                   └── clothing_store/
│                       ├── controller/
│                       ├── service/
│                       └── ClothingStoreApplicationTests.java
└── build.gradle

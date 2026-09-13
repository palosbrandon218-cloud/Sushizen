package com.sushizen.app

import android.os.Bundle
import androidx.activity.ComponentActivity
import androidx.activity.compose.setContent
import androidx.activity.viewModels
import androidx.compose.foundation.background
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.foundation.shape.RoundedCornerShape
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.History
import androidx.compose.material.icons.filled.Home
import androidx.compose.material.icons.filled.RestaurantMenu
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.graphics.Color
import androidx.compose.ui.text.font.FontWeight
import androidx.compose.ui.text.style.TextAlign
import androidx.compose.ui.unit.dp
import androidx.compose.ui.unit.sp
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.asStateFlow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.launch

// ==========================================
// 1. DATA LAYER (MODELOS Y REPOSITORIO)
// ==========================================

data class SushiItem(
    val id: String,
    val nameEs: String,
    val nameEn: String,
    val descriptionEs: String,
    val descriptionEn: String,
    val price: Double,
    val emoji: String
)

class SushiRepository {
    fun getMenu(): Flow<List<SushiItem>> = flow {
        delay(1200) // Simular carga de red
        emit(
            listOf(
                SushiItem("1", "Roll California 🍣", "California Roll 🍣", "Cangrejo, aguacate y pepino", "Crab, avocado, and cucumber", 120.0, "🍣"),
                SushiItem("2", "Nigiri de Salmón 🐟", "Salmon Nigiri 🐟", "Salmón fresco sobre arroz sazonado", "Fresh salmon over seasoned rice", 95.0, "🍣"),
                SushiItem("3", "Roll Dragón Picante 🐉", "Spicy Dragon Roll 🐉", "Anguila, aguacate y salsa spicy", "Eel, avocado, and spicy sauce", 150.0, "🍱"),
                SushiItem("4", "Tempura Mixto 🍤", "Mixed Tempura 🍤", "Camarón y verduras empanizadas", "Shrimp and fried vegetables", 110.0, "🍤")
            )
        )
    }
}

// ==========================================
// 2. STATE & VIEWMODEL
// ==========================================

sealed interface UiState<out T> {
    object Loading : UiState<Nothing>
    data class Success<T>(val data: T) : UiState<T>
    data class Error(val message: String) : UiState<Nothing>
}

enum class AppLanguage { ES, EN }

class SushiViewModel : ViewModel() {
    private val repository = SushiRepository()

    private val _menuState = MutableStateFlow<UiState<List<SushiItem>>>(UiState.Loading)
    val menuState: StateFlow<UiState<List<SushiItem>>> = _menuState.asStateFlow()

    private val _currentLanguage = MutableStateFlow(AppLanguage.ES)
    val currentLanguage: StateFlow<AppLanguage> = _currentLanguage.asStateFlow()

    init {
        fetchMenu()
    }

    fun fetchMenu() {
        viewModelScope.launch {
            _menuState.value = UiState.Loading
            repository.getMenu()
                .catch { e -> _menuState.value = UiState.Error(e.localizedMessage ?: "Error") }
                .collect { items -> _menuState.value = UiState.Success(items) }
        }
    }

    fun toggleLanguage() {
        _currentLanguage.value = if (_currentLanguage.value == AppLanguage.ES) AppLanguage.EN else AppLanguage.ES
    }
}

// ==========================================
// 3. MAIN ACTIVITY & ENTRY POINT
// ==========================================

class MainActivity : ComponentActivity() {
    private val viewModel: SushiViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MaterialTheme {
                SushiZenApp(viewModel)
            }
        }
    }
}

@Composable
fun SushiZenApp(viewModel: SushiViewModel) {
    val navController = rememberNavController()
    val menuState by viewModel.menuState.collectAsState()
    val language by viewModel.currentLanguage.collectAsState()

    Scaffold(
        bottomBar = {
            NavigationBar {
                NavigationBarItem(
                    icon = { Icon(Icons.Default.Home, contentDescription = "Inicio") },
                    label = { Text(if (language == AppLanguage.ES) "Inicio" else "Home") },
                    selected = false,
                    onClick = { navController.navigate("welcome") }
                )
                NavigationBarItem(
                    icon = { Icon(Icons.Default.RestaurantMenu, contentDescription = "Menú") },
                    label = { Text(if (language == AppLanguage.ES) "Menú" else "Menu") },
                    selected = false,
                    onClick = { navController.navigate("menu") }
                )
                NavigationBarItem(
                    icon = { Icon(Icons.Default.History, contentDescription = "Ajustes") },
                    label = { Text(if (language == AppLanguage.ES) "Ajustes" else "Settings") },
                    selected = false,
                    onClick = { navController.navigate("history") }
                )
            }
        }
    ) { paddingValues ->
        NavHost(
            navController = navController,
            startDestination = "welcome",
            modifier = Modifier.padding(paddingValues)
        ) {
            composable("welcome") {
                WelcomeScreen(
                    language = language,
                    onNavigateToMenu = { navController.navigate("menu") }
                )
            }
            composable("menu") {
                MenuScreen(
                    menuState = menuState,
                    language = language,
                    onRetry = { viewModel.fetchMenu() }
                )
            }
            composable("history") {
                OrderHistoryScreen(
                    currentLanguage = language,
                    onToggleLanguage = { viewModel.toggleLanguage() }
                )
            }
        }
    }
}

// ==========================================
// 4. UI SCREENS (PANTALLAS)
// ==========================================

@Composable
fun WelcomeScreen(
    language: AppLanguage,
    onNavigateToMenu: () -> Unit
) {
    Column(
        modifier = Modifier
            .fillMaxSize()
            .background(MaterialTheme.colorScheme.background)
            .padding(24.dp),
        horizontalAlignment = Alignment.CenterHorizontally,
        verticalArrangement = Arrangement.SpaceBetween
    ) {
        Spacer(modifier = Modifier.height(20.dp))

        Column(horizontalAlignment = Alignment.CenterHorizontally) {
            Text(text = "🍣 (っ˘ڡ˘ς) 🍱", fontSize = 54.sp)
            Spacer(modifier = Modifier.height(16.dp))
            Text(
                text = "Sushi Zen",
                fontSize = 36.sp,
                fontWeight = FontWeight.Bold,
                color = MaterialTheme.colorScheme.primary
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = if (language == AppLanguage.ES) 
                    "¡El sushi más fresco directo a tu puerta!" 
                else 
                    "Fresh sushi delivered straight to your door!",
                fontSize = 16.sp,
                textAlign = TextAlign.Center,
                color = MaterialTheme.colorScheme.onBackground.copy(alpha = 0.7f)
            )
        }

        Button(
            onClick = onNavigateToMenu,
            modifier = Modifier
                .fillMaxWidth()
                .height(56.dp),
            shape = RoundedCornerShape(16.dp),
            colors = ButtonDefaults.buttonColors(containerColor = Color(0xFFFF5722))
        ) {
            Text(
                text = if (language == AppLanguage.ES) "Ordenar Ahora" else "Order Now",
                fontSize = 18.sp,
                color = Color.White
            )
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun MenuScreen(
    menuState: UiState<List<SushiItem>>,
    language: AppLanguage,
    onRetry: () -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(if (language == AppLanguage.ES) "Menú Sushi Zen" else "Sushi Zen Menu") }
            )
        }
    ) { paddingValues ->
        Box(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
        ) {
            when (menuState) {
                is UiState.Loading -> {
                    CircularProgressIndicator(modifier = Modifier.align(Alignment.Center))
                }
                is UiState.Error -> {
                    Column(
                        modifier = Modifier.align(Alignment.Center),
                        horizontalAlignment = Alignment.CenterHorizontally
                    ) {
                        Text(text = "❌ Error: ${menuState.message}")
                        Spacer(modifier = Modifier.height(8.dp))
                        Button(onClick = onRetry) {
                            Text(if (language == AppLanguage.ES) "Reintentar" else "Retry")
                        }
                    }
                }
                is UiState.Success -> {
                    LazyColumn(
                        contentPadding = PaddingValues(16.dp),
                        verticalArrangement = Arrangement.spacedBy(12.dp)
                    ) {
                        items(menuState.data) { item ->
                            SushiCard(item = item, language = language)
                        }
                    }
                }
            }
        }
    }
}

@Composable
fun SushiCard(item: SushiItem, language: AppLanguage) {
    Card(
        modifier = Modifier.fillMaxWidth(),
        shape = RoundedCornerShape(12.dp),
        colors = CardDefaults.cardColors(containerColor = MaterialTheme.colorScheme.surfaceVariant)
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Text(text = item.emoji, fontSize = 40.sp)
            Spacer(modifier = Modifier.width(16.dp))
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = if (language == AppLanguage.ES) item.nameEs else item.nameEn,
                    fontWeight = FontWeight.Bold,
                    fontSize = 16.sp
                )
                Text(
                    text = if (language == AppLanguage.ES) item.descriptionEs else item.descriptionEn,
                    fontSize = 12.sp,
                    color = MaterialTheme.colorScheme.onSurfaceVariant.copy(alpha = 0.7f)
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    text = "$${item.price} MXN",
                    fontWeight = FontWeight.SemiBold,
                    color = Color(0xFFFF5722)
                )
            }
            Button(onClick = { /* Pedir */ }) {
                Text(if (language == AppLanguage.ES) "Pedir" else "Add")
            }
        }
    }
}

@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun OrderHistoryScreen(
    currentLanguage: AppLanguage,
    onToggleLanguage: () -> Unit
) {
    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text(if (currentLanguage == AppLanguage.ES) "Ajustes e Historial" else "Settings & History") }
            )
        }
    ) { paddingValues ->
        Column(
            modifier = Modifier
                .fillMaxSize()
                .padding(paddingValues)
                .padding(16.dp)
        ) {
            Card(modifier = Modifier.fillMaxWidth()) {
                Row(
                    modifier = Modifier
                        .fillMaxWidth()
                        .padding(16.dp),
                    horizontalArrangement = Arrangement.SpaceBetween,
                    verticalAlignment = Alignment.CenterVertically
                ) {
                    Text(
                        text = if (currentLanguage == AppLanguage.ES) "Idioma / Language" else "Language / Idioma",
                        fontSize = 16.sp
                    )
                    Button(onClick = onToggleLanguage) {
                        Text(if (currentLanguage == AppLanguage.ES) "Español (ES)" else "English (EN)")
                    }
                }
            }

            Spacer(modifier = Modifier.height(24.dp))

            Text(
                text = if (currentLanguage == AppLanguage.ES) "Historial de Pedidos" else "Order History",
                fontSize = 20.sp,
                fontWeight = FontWeight.Bold
            )
            Spacer(modifier = Modifier.height(8.dp))
            Text(
                text = if (currentLanguage == AppLanguage.ES) "No hay pedidos anteriores." else "No previous orders.",
                color = MaterialTheme.colorScheme.onBackground.copy(alpha = 0.5f)
            )
        }
    }
}

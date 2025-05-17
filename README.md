# burs_Oyunu


use soroban_sdk::{contract, contractimpl, Address, Env, String, Symbol, Vec, Map, FromVal, IntoVal};
use soroban_token_sdk::{TokenUtils, metadata::TokenMetadata};

// Sabitler
const ADMIN_KEY: Symbol = Symbol::short("admin");
const META_KEY: Symbol = Symbol::short("meta");
const SCHOLARSHIP_POOL_KEY: Symbol = Symbol::short("schlpool");
const QUEST_COUNTER_KEY: Symbol = Symbol::short("qstcntr");
const SCHOLARSHIP_REQ_COUNTER_KEY: Symbol = Symbol::short("sreqcntr");
const POINTS_KEY: Symbol = Symbol::short("points");
const USER_LEVEL_KEY: Symbol = Symbol::short("userlvl");
const USER_BADGES_KEY: Symbol = Symbol::short("badges");

// Veri Yapıları
#[derive(Clone)]
pub struct ScholarshipRequest {
    pub id: u32,
    pub student: Address,
    pub amount: i128,
    pub description: String,
    pub collected: i128,
    pub completed: bool,
}

#[derive(Clone)]
pub struct Quest {
    pub id: u32,
    pub title: String,
    pub description: String,
    pub reward_points: u32,
    pub reward_tokens: i128,
    pub active: bool,
}

#[derive(Clone)]
pub struct UserStats {
    pub level: u32,
    pub total_points: u32,
    pub donations_made: i128,
    pub quests_completed: u32,
}

#[derive(Clone)]
pub struct Badge {
    pub id: u32,
    pub name: String,
    pub description: String,
}

// Sözleşme Yapısı
#[contract]
pub struct EduQuest;

#[contractimpl]
impl EduQuest {
    // ========== BAŞLATMA FONKSİYONU ==========
    pub fn initialize(e: Env, admin: Address, decimal: u32, name: String, symbol: String) {
        // Admin'i kaydet
        e.storage().instance().set(&ADMIN_KEY, &admin);
        
        // Token meta verilerini kaydet
        e.storage().instance().set(
            &META_KEY,
            &TokenMetadata {
                decimal,
                name,
                symbol,
            },
        );
        
        // Burs havuzunu başlat
        e.storage().instance().set(&SCHOLARSHIP_POOL_KEY, &0_i128);
        
        // Sayaçları başlat
        e.storage().instance().set(&QUEST_COUNTER_KEY, &0_u32);
        e.storage().instance().set(&SCHOLARSHIP_REQ_COUNTER_KEY, &0_u32);
    }

    // ========== TOKEN FONKSİYONLARI ==========
    // Token üretimi - sadece admin yapabilir
    pub fn mint(e: Env, to: Address, amount: i128) {
        let admin: Address = e.storage().instance().get(&ADMIN_KEY).unwrap();
        admin.require_auth();

        let mut balance: i128 = e.storage().instance().get(&to).unwrap_or(0);
        balance += amount;
        e.storage().instance().set(&to, &balance);

        TokenUtils::new(&e).events().mint(admin, to, amount);
    }

    // Token transferi
    pub fn transfer(e: Env, from: Address, to: Address, amount: i128) {
        from.require_auth();

        let mut from_balance: i128 = e.storage().instance().get(&from).unwrap_or(0);
        let mut to_balance: i128 = e.storage().instance().get(&to).unwrap_or(0);

        if from_balance < amount {
            panic!("Yetersiz bakiye");
        }

        from_balance -= amount;
        to_balance += amount;
        e.storage().instance().set(&from, &from_balance);
        e.storage().instance().set(&to, &to_balance);

        TokenUtils::new(&e).events().transfer(from, to, amount);
    }

    // Bakiye sorgulama
    pub fn balance(e: Env, id: Address) -> i128 {
        e.storage().instance().get(&id).unwrap_or(0)
    }

    // ========== BURS TALEBİ FONKSİYONLARI ==========
    // Yeni burs talebi oluştur
    pub fn create_scholarship_request(
        e: Env, 
        student: Address, 
        amount: i128, 
        description: String
    ) -> u32 {
        student.require_auth();
        
        // Yeni ID oluştur
        let mut request_counter: u32 = e.storage().instance().get(&SCHOLARSHIP_REQ_COUNTER_KEY).unwrap();
        request_counter += 1;
        
        // Burs talebini oluştur
        let request = ScholarshipRequest {
            id: request_counter,
            student: student.clone(),
            amount,
            description,
            collected: 0,
            completed: false,
        };
        
        // Talebi sakla
        let key = Symbol::short(&format!("sreq{}", request_counter));
        e.storage().instance().set(&key, &request);
        
        // Sayacı güncelle
        e.storage().instance().set(&SCHOLARSHIP_REQ_COUNTER_KEY, &request_counter);
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("new_request"), student);
        event.publish(topics, request_counter);
        
        request_counter
    }
    
    // Burs talebine bağış yap
    pub fn donate_to_request(e: Env, donor: Address, request_id: u32, amount: i128) {
        donor.require_auth();
        
        // Donör bakiyesini kontrol et
        let donor_balance: i128 = e.storage().instance().get(&donor).unwrap_or(0);
        if donor_balance < amount {
            panic!("Yetersiz bakiye");
        }
        
        // Burs talebini getir
        let key = Symbol::short(&format!("sreq{}", request_id));
        let mut request: ScholarshipRequest = e.storage().instance().get(&key).unwrap();
        
        if request.completed {
            panic!("Bu burs talebi tamamlandı");
        }
        
        // Fonları transfer et
        let mut donor_balance = e.storage().instance().get(&donor).unwrap_or(0);
        donor_balance -= amount;
        e.storage().instance().set(&donor, &donor_balance);
        
        // Öğrenciye transfer et
        let mut student_balance = e.storage().instance().get(&request.student).unwrap_or(0);
        student_balance += amount;
        e.storage().instance().set(&request.student, &student_balance);
        
        // Toplanan miktarı güncelle
        request.collected += amount;
        
        // Burs talebi tamamlandı mı kontrol et
        if request.collected >= request.amount {
            request.completed = true;
        }
        
        // Burs talebini güncelle
        e.storage().instance().set(&key, &request);
        
        // Puan ver
        Self::award_points(e, donor.clone(), amount / 10);  // 10 XLM = 1 puan
        
        // Rozet kontrol et
        Self::check_and_award_badges(e, donor.clone());
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("donation"), donor, request.student);
        event.publish(topics, amount);
    }
    
    // Burs talep detaylarını getir
    pub fn get_scholarship_request(e: Env, request_id: u32) -> ScholarshipRequest {
        let key = Symbol::short(&format!("sreq{}", request_id));
        e.storage().instance().get(&key).unwrap()
    }
    
    // Tüm burs taleplerini listele
    pub fn list_scholarship_requests(e: Env) -> Vec<ScholarshipRequest> {
        let total = e.storage().instance().get(&SCHOLARSHIP_REQ_COUNTER_KEY).unwrap_or(0);
        let mut requests = Vec::new(&e);
        
        for i in 1..=total {
            let key = Symbol::short(&format!("sreq{}", i));
            if let Some(request) = e.storage().instance().get(&key) {
                requests.push_back(request);
            }
        }
        
        requests
    }

    // ========== GÖREV SİSTEMİ FONKSİYONLARI ==========
    // Yeni görev oluştur (admin işlemi)
    pub fn create_quest(
        e: Env,
        admin: Address,
        title: String,
        description: String,
        reward_points: u32,
        reward_tokens: i128
    ) -> u32 {
        // Admin yetkisini kontrol et
        let stored_admin: Address = e.storage().instance().get(&ADMIN_KEY).unwrap();
        if admin != stored_admin {
            panic!("Sadece admin görev oluşturabilir");
        }
        admin.require_auth();
        
        // Yeni ID oluştur
        let mut quest_counter: u32 = e.storage().instance().get(&QUEST_COUNTER_KEY).unwrap();
        quest_counter += 1;
        
        // Görevi oluştur
        let quest = Quest {
            id: quest_counter,
            title,
            description,
            reward_points,
            reward_tokens,
            active: true,
        };
        
        // Görevi sakla
        let key = Symbol::short(&format!("quest{}", quest_counter));
        e.storage().instance().set(&key, &quest);
        
        // Sayacı güncelle
        e.storage().instance().set(&QUEST_COUNTER_KEY, &quest_counter);
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("new_quest"));
        event.publish(topics, quest_counter);
        
        quest_counter
    }
    
    // Görevi tamamla
    pub fn complete_quest(e: Env, user: Address, quest_id: u32) {
        user.require_auth();
        
        // Görevi getir
        let key = Symbol::short(&format!("quest{}", quest_id));
        let quest: Quest = e.storage().instance().get(&key).unwrap();
        
        if !quest.active {
            panic!("Bu görev artık aktif değil");
        }
        
        // Tamamlanma durumunu kontrol et
        let completion_key = Symbol::short(&format!("qcmp{}_{}", quest_id, user.to_string()));
        let already_completed: bool = e.storage().instance().get(&completion_key).unwrap_or(false);
        
        if already_completed {
            panic!("Bu görevi zaten tamamladınız");
        }
        
        // Token ödülü ver
        if quest.reward_tokens > 0 {
            // Admin'den kullanıcıya ödül tokenları transfer et
            let admin: Address = e.storage().instance().get(&ADMIN_KEY).unwrap();
            let mut admin_balance: i128 = e.storage().instance().get(&admin).unwrap_or(0);
            let mut user_balance: i128 = e.storage().instance().get(&user).unwrap_or(0);
            
            admin_balance -= quest.reward_tokens;
            user_balance += quest.reward_tokens;
            
            e.storage().instance().set(&admin, &admin_balance);
            e.storage().instance().set(&user, &user_balance);
            
            // Token transfer olayını kaydet
            TokenUtils::new(&e).events().transfer(admin, user.clone(), quest.reward_tokens);
        }
        
        // Puan ödülü ver
        Self::award_points(e, user.clone(), quest.reward_points as i128);
        
        // Tamamlanma durumunu güncelle
        e.storage().instance().set(&completion_key, &true);
        
        // Kullanıcının tamamladığı görev sayısını güncelle
        let user_stats_key = Symbol::short(&format!("stats_{}", user.to_string()));
        let mut user_stats: UserStats = e.storage().instance().get(&user_stats_key).unwrap_or(
            UserStats {
                level: 1,
                total_points: 0,
                donations_made: 0,
                quests_completed: 0,
            }
        );
        
        user_stats.quests_completed += 1;
        e.storage().instance().set(&user_stats_key, &user_stats);
        
        // Rozet kontrol et
        Self::check_and_award_badges(e, user.clone());
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("quest_completed"), user);
        event.publish(topics, quest_id);
    }
    
    // Görev detaylarını getir
    pub fn get_quest(e: Env, quest_id: u32) -> Quest {
        let key = Symbol::short(&format!("quest{}", quest_id));
        e.storage().instance().get(&key).unwrap()
    }
    
    // Tüm görevleri listele
    pub fn list_quests(e: Env) -> Vec<Quest> {
        let total = e.storage().instance().get(&QUEST_COUNTER_KEY).unwrap_or(0);
        let mut quests = Vec::new(&e);
        
        for i in 1..=total {
            let key = Symbol::short(&format!("quest{}", i));
            if let Some(quest) = e.storage().instance().get(&key) {
                quests.push_back(quest);
            }
        }
        
        quests
    }

    // ========== BURS HAVUZU FONKSİYONLARI ==========
    // Burs havuzuna bağış yap
    pub fn donate_to_pool(e: Env, donor: Address, amount: i128) {
        donor.require_auth();
        
        // Bakiyeyi kontrol et
        let mut donor_balance: i128 = e.storage().instance().get(&donor).unwrap_or(0);
        if donor_balance < amount {
            panic!("Yetersiz bakiye");
        }
        
        // Havuz bakiyesini güncelle
        let mut pool_balance: i128 = e.storage().instance().get(&SCHOLARSHIP_POOL_KEY).unwrap();
        
        // Fonları transfer et
        donor_balance -= amount;
        pool_balance += amount;
        
        e.storage().instance().set(&donor, &donor_balance);
        e.storage().instance().set(&SCHOLARSHIP_POOL_KEY, &pool_balance);
        
        // Puan ver
        Self::award_points(e, donor.clone(), amount / 5);  // Havuza bağış daha fazla puan kazandırır
        
        // Kullanıcının yaptığı bağış miktarını güncelle
        let user_stats_key = Symbol::short(&format!("stats_{}", donor.to_string()));
        let mut user_stats: UserStats = e.storage().instance().get(&user_stats_key).unwrap_or(
            UserStats {
                level: 1,
                total_points: 0,
                donations_made: 0,
                quests_completed: 0,
            }
        );
        
        user_stats.donations_made += amount;
        e.storage().instance().set(&user_stats_key, &user_stats);
        
        // Rozet kontrol et
        Self::check_and_award_badges(e, donor.clone());
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("pool_donation"), donor);
        event.publish(topics, amount);
    }
    
    // Havuzdan burs ver (admin işlemi)
    pub fn award_scholarship_from_pool(e: Env, admin: Address, student: Address, amount: i128, reason: String) {
        // Admin yetkisini kontrol et
        let stored_admin: Address = e.storage().instance().get(&ADMIN_KEY).unwrap();
        if admin != stored_admin {
            panic!("Sadece admin burs verebilir");
        }
        admin.require_auth();
        
        // Havuz bakiyesini kontrol et
        let mut pool_balance: i128 = e.storage().instance().get(&SCHOLARSHIP_POOL_KEY).unwrap();
        if pool_balance < amount {
            panic!("Havuzda yeterli bakiye yok");
        }
        
        // Öğrenciye burs ver
        let mut student_balance: i128 = e.storage().instance().get(&student).unwrap_or(0);
        
        pool_balance -= amount;
        student_balance += amount;
        
        e.storage().instance().set(&SCHOLARSHIP_POOL_KEY, &pool_balance);
        e.storage().instance().set(&student, &student_balance);
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("scholarship_awarded"), student);
        let data = (amount, reason);
        event.publish(topics, data);
    }
    
    // Havuz bakiyesini getir
    pub fn get_scholarship_pool_balance(e: Env) -> i128 {
        e.storage().instance().get(&SCHOLARSHIP_POOL_KEY).unwrap_or(0)
    }

    // ========== PUAN VE OYUNLAŞTIRMA FONKSİYONLARI ==========
    // Puan ver (iç fonksiyon)
    fn award_points(e: Env, user: Address, points: i128) {
        let user_points_key = Symbol::short(&format!("points_{}", user.to_string()));
        let mut current_points: u32 = e.storage().instance().get(&user_points_key).unwrap_or(0);
        current_points += points as u32;
        e.storage().instance().set(&user_points_key, &current_points);
        
        // Kullanıcı istatistiklerini güncelle
        let user_stats_key = Symbol::short(&format!("stats_{}", user.to_string()));
        let mut user_stats: UserStats = e.storage().instance().get(&user_stats_key).unwrap_or(
            UserStats {
                level: 1,
                total_points: 0,
                donations_made: 0,
                quests_completed: 0,
            }
        );
        
        user_stats.total_points += points as u32;
        
        // Seviye kontrolü
        user_stats.level = 1 + (user_stats.total_points / 100); // Her 100 puan bir seviye
        
        e.storage().instance().set(&user_stats_key, &user_stats);
        
        // Olayı kaydet
        let event = TokenUtils::new(&e).events();
        let topics = (Symbol::short("points_awarded"), user);
        event.publish(topics, points);
    }
    
    // Rozet kontrol ve verme (iç fonksiyon)
    fn check_and_award_badges(e: Env, user: Address) {
        // Kullanıcı istatistiklerini al
        let user_stats_key = Symbol::short(&format!("stats_{}", user.to_string()));
        let user_stats: UserStats = e.storage().instance().get(&user_stats_key).unwrap_or(
            UserStats {
                level: 1,
                total_points: 0,
                donations_made: 0,
                quests_completed: 0,
            }
        );
        
        // Kullanıcının rozetlerini al
        let user_badges_key = Symbol::short(&format!("badges_{}", user.to_string()));
        let mut user_badges: Vec<u32> = e.storage().instance().get(&user_badges_key).unwrap_or(Vec::new(&e));
        
        // Rozet kontrolü
        let mut new_badges = Vec::new(&e);
        
        // Örnek rozetler
        if user_stats.donations_made >= 1000 && !user_badges.contains(&1) {
            new_badges.push_back(1); // Cömert Bağışçı Rozeti
            user_badges.push_back(1);
        }
        
        if user_stats.quests_completed >= 10 && !user_badges.contains(&2) {
            new_badges.push_back(2); // Görev Uzmanı Rozeti
            user_badges.push_back(2);
        }
        
        if user_stats.level >= 5 && !user_badges.contains(&3) {
            new_badges.push_back(3); // İleri Seviye Rozeti
            user_badges.push_back(3);
        }
        
        // Yeni rozetler varsa sakla
        if !new_badges.is_empty() {
            e.storage().instance().set(&user_badges_key, &user_badges);
            
            // Olayı kaydet
            let event = TokenUtils::new(&e).events();
            let topics = (Symbol::short("badges_awarded"), user);
            event.publish(topics, new_badges);
        }
    }
    
    // Kullanıcı puanını getir
    pub fn get_user_points(e: Env, user: Address) -> u32 {
        let user_points_key = Symbol::short(&format!("points_{}", user.to_string()));
        e.storage().instance().get(&user_points_key).unwrap_or(0)
    }
    
    // Kullanıcı seviyesini getir
    pub fn get_user_level(e: Env, user: Address) -> u32 {
        let user_stats_key = Symbol::short(&format!("stats_{}", user.to_string()));
        let user_stats: UserStats = e.storage().instance().get(&user_stats_key).unwrap_or(
            UserStats {
                level: 1,
                total_points: 0,
                donations_made: 0,
                quests_completed: 0,
            }
        );
        
        user_stats.level
    }
    
    // Kullanıcı rozetlerini getir
    pub fn get_user_badges(e: Env, user: Address) -> Vec<Badge> {
        let user_badges_key = Symbol::short(&format!("badges_{}", user.to_string()));
        let badge_ids: Vec<u32> = e.storage().instance().get(&user_badges_key).unwrap_or(Vec::new(&e));
        
        let mut badges = Vec::new(&e);
        
        for id in badge_ids.iter() {
            match id {
                1 => badges.push_back(Badge {
                    id: 1,
                    name: String::from_str(&e, "Cömert Bağışçı"),
                    description: String::from_str(&e, "1000 XLM'den fazla bağış yaptı"),
                }),
                2 => badges.push_back(Badge {
                    id: 2,
                    name: String::from_str(&e, "Görev Uzmanı"),
                    description: String::from_str(&e, "10'dan fazla görev tamamladı"),
                }),
                3 => badges.push_back(Badge {
                    id: 3,
                    name: String::from_str(&e, "İleri Seviye"),
                    description: String::from_str(&e, "Seviye 5'e ulaştı"),
                }),
                _ => {}
            }
        }
        
        badges
    }
    
    // Kullanıcı istatistiklerini getir
    pub fn get_user_stats(e: Env, user: Address) -> UserStats {
        let user_stats_key = Symbol::short(&format!("stats_{}", user.to_string()));
        e.storage().instance().get(&user_stats_key).unwrap_or(
            UserStats {
                level: 1,
                total_points: 0,
                donations_made: 0,
                quests_completed: 0,
            }
        )
    }
    
    // Liderlik tablosu - en çok puan toplayan kullanıcılar
    pub fn get_leaderboard(e: Env, limit: u32) -> Vec<(Address, u32)> {
        // Not: Gerçek bir liderlik tablosu için daha karmaşık bir yapı gerekir
        // Bu basit bir örnek implementasyondur
        
        // Tüm kullanıcıları ve puanlarını topla (gerçek uygulamada daha verimli bir veri yapısı kullanılmalı)
        let mut user_points = Vec::new(&e);
        
        // Burada örnek olarak yalnızca belirli bilinen kullanıcıları kontrol ediyoruz
        // Gerçek uygulamada daha iyi bir kullanıcı takip mekanizması olmalıdır
        
        // Sözleşme ile etkileşime giren tüm adresleri elde etmek için ek bir mekanizma gerekir
        // Bu örnekte sınırlı bir yaklaşım gösteriyoruz
        
        // Burada daha fazla kod olacak...
        
        // Placeholder olarak boş bir liste dönüyoruz
        user_points
    }
}

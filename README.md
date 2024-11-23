import Debug "mo:base/Debug";
import Float "mo:base/Float";
import Int "mo:base/Int";
import Array "mo:base/Array";
import Text "mo:base/Text";
import Buffer "mo:base/Buffer";
import HashMap "mo:base/HashMap";
import Iter "mo:base/Iter";
import Time "mo:base/Time";

actor TaxCalculator {
    // Veri Yapıları
    type Income = {
        salary: Float; // Maaş geliri
        rental: Float; // Kira geliri
        investment: Float; // Yatırım geliri
        business: Float; // Ticari kazanç
        other: Float; // Diğer gelirler
    };

    type Expense = {
        education: Float; // Eğitim harcamaları
        health: Float; // Sağlık harcamaları
        housing: Float; // Konut harcamaları
        donations: Float; // Bağışlar
        other: Float; // Diğer giderler
    };

    type TaxDeduction = {
        type_: Text;
        amount: Float;
        description: Text;
    };

    type TaxReport = {
        year: Nat;
        totalIncome: Float;
        totalDeductions: Float;
        taxableIncome: Float;
        calculatedTax: Float;
        effectiveTaxRate: Float;
        deductions: [TaxDeduction];
        recommendations: [Text];
    };

    // Vergi dilimleri (2024 yılı için örnek)
    private let taxBrackets = [
        { limit = 70000.0; rate = 0.15 },
        { limit = 150000.0; rate = 0.20 },
        { limit = 370000.0; rate = 0.27 },
        { limit = 880000.0; rate = 0.35 },
        { limit = 880000.1; rate = 0.40 }
    ];

    // Kullanıcı verileri için storage
    private var userIncomes = HashMap.HashMap<Text, Income>(0, Text.equal, Text.hash);
    private var userExpenses = HashMap.HashMap<Text, Expense>(0, Text.equal, Text.hash);
    private var userReports = HashMap.HashMap<Text, TaxReport>(0, Text.equal, Text.hash);

    // Gelir bilgilerini kaydetme
    public func updateIncome(userId: Text, income: Income) : async Text {
        userIncomes.put(userId, income);
        return "Gelir bilgileri güncellendi";
    };

    // Gider bilgilerini kaydetme
    public func updateExpense(userId: Text, expense: Expense) : async Text {
        userExpenses.put(userId, expense);
        return "Gider bilgileri güncellendi";
    };

    // Toplam gelir hesaplama
    private func calculateTotalIncome(income: Income) : Float {
        return income.salary + 
               income.rental + 
               income.investment + 
               income.business + 
               income.other;
    };

    // Vergi indirimlerini hesaplama
    private func calculateDeductions(expense: Expense) : [TaxDeduction] {
        var deductions = Buffer.Buffer<TaxDeduction>(0);

        // Eğitim harcamaları indirimi
        if (expense.education > 0) {
            deductions.add({
                type_ = "Eğitim";
                amount = Float.min(expense.education, 2000.0); // Maximum 2000 TL
                description = "Eğitim harcamaları indirimi";
            });
        };

        // Sağlık harcamaları indirimi
        if (expense.health > 0) {
            deductions.add({
                type_ = "Sağlık";
                amount = expense.health * 0.15; // %15 indirim
                description = "Sağlık harcamaları indirimi";
            });
        };

        // Konut kredisi faiz indirimi
        if (expense.housing > 0) {
            deductions.add({
                type_ = "Konut";
                amount = Float.min(expense.housing * 0.10, 10000.0);
                description = "Konut kredisi faiz indirimi";
            });
        };

        // Bağış indirimi
        if (expense.donations > 0) {
            deductions.add({
                type_ = "Bağış";
                amount = expense.donations;
                description = "Bağış indirimi";
            });
        };

        return Buffer.toArray(deductions);
    };

    // Gelir vergisi hesaplama
    private func calculateIncomeTax(taxableIncome: Float) : Float {
        var remainingIncome = taxableIncome;
        var totalTax : Float = 0;
        var previousLimit : Float = 0;

        for (bracket in taxBrackets.vals()) {
            if (remainingIncome > 0) {
                let taxableAmount = Float.min(remainingIncome, bracket.limit - previousLimit);
                totalTax += taxableAmount * bracket.rate;
                remainingIncome -= taxableAmount;
                previousLimit := bracket.limit;
            };
        };

        return totalTax;
    };

    // Vergi optimizasyonu önerileri
    private func generateTaxRecommendations(income: Income, expense: Expense) : [Text] {
        var recommendations = Buffer.Buffer<Text>(0);

        // Gelir bazlı öneriler
        if (income.investment > 0) {
            recommendations.add("Uzun vadeli yatırım araçlarını tercih ederek vergi avantajından yararlanabilirsiniz.");
        };

        // Gider bazlı öneriler
        if (expense.education == 0) {
            recommendations.add("Eğitim harcamalarınızı belgelendirerek vergi indirimi alabilirsiniz.");
        };

        if (expense.health == 0) {
            recommendations.add("Sağlık harcamalarınız için fatura almanız durumunda vergi indirimi mümkündür.");
        };

        if (expense.donations == 0) {
            recommendations.add("Vergi muafiyeti olan kurumlara yapacağınız bağışlar vergi matrahınızı düşürebilir.");
        };

        return Buffer.toArray(recommendations);
    };

    // Vergi raporu oluşturma
    public func generateTaxReport(userId: Text, year: Nat) : async ?TaxReport {
        switch (userIncomes.get(userId), userExpenses.get(userId)) {
            case (?income, ?expense) {
                let totalIncome = calculateTotalIncome(income);
                let deductions = calculateDeductions(expense);
                let totalDeductions = Array.foldLeft<TaxDeduction, Float>(
                    deductions,
                    0,
                    func(acc, deduction) { acc + deduction.amount }
                );

                let taxableIncome = Float.max(0, totalIncome - totalDeductions);
                let calculatedTax = calculateIncomeTax(taxableIncome);
                let effectiveTaxRate = if (totalIncome > 0) {
                    calculatedTax / totalIncome
                } else {
                    0.0
                };

                let recommendations = generateTaxRecommendations(income, expense);

                let report : TaxReport = {
                    year = year;
                    totalIncome = totalIncome;
                    totalDeductions = totalDeductions;
                    taxableIncome = taxableIncome;
                    calculatedTax = calculatedTax;
                    effectiveTaxRate = effectiveTaxRate;
                    deductions = deductions;
                    recommendations = recommendations;
                };

                userReports.put(userId, report);
                return ?report;
            };
            case _ { return null };
        };
    };

    // KDV hesaplama (Oran bazlı)
    public func calculateVAT(amount: Float, rate: Float) : async Float {
        return amount * (rate / 100.0);
    };

    // Vergi özeti alma
    public query func getTaxSummary(userId: Text) : async ?{
        lastReport: TaxReport;
        vatTotal: Float;
        pendingDeductions: Float;
    } {
        switch (userReports.get(userId)) {
            case (?report) {
                return ?{
                    lastReport = report;
                    vatTotal = 0.0; // Bu değer gerçek sistemde KDV toplamından gelecek
                    pendingDeductions = report.totalDeductions;
                };
            };
            case null { return null };
        };
    };
}

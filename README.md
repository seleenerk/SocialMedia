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
    
    type Income = {
        salary: Float; 
        rental: Float; 
        investment: Float; 
        business: Float; 
        other: Float; 
    };

    type Expense = {
        education: Float; 
        health: Float; 
        housing: Float; 
        donations: Float; 
        other: Float; 
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

    
    private let taxBrackets = [
        { limit = 70000.0; rate = 0.15 },
        { limit = 150000.0; rate = 0.20 },
        { limit = 370000.0; rate = 0.27 },
        { limit = 880000.0; rate = 0.35 },
        { limit = 880000.1; rate = 0.40 }
    ];

    private var userIncomes = HashMap.HashMap<Text, Income>(0, Text.equal, Text.hash);
    private var userExpenses = HashMap.HashMap<Text, Expense>(0, Text.equal, Text.hash);
    private var userReports = HashMap.HashMap<Text, TaxReport>(0, Text.equal, Text.hash);

    public func updateIncome(userId: Text, income: Income) : async Text {
        userIncomes.put(userId, income);
        return "Gelir bilgileri güncellendi";
    };

    public func updateExpense(userId: Text, expense: Expense) : async Text {
        userExpenses.put(userId, expense);
        return "Gider bilgileri güncellendi";
    };

    private func calculateTotalIncome(income: Income) : Float {
        return income.salary + 
               income.rental + 
               income.investment + 
               income.business + 
               income.other;
    };

    private func calculateDeductions(expense: Expense) : [TaxDeduction] {
        var deductions = Buffer.Buffer<TaxDeduction>(0);

        if (expense.education > 0) {
            deductions.add({
                type_ = "Eğitim";
                amount = Float.min(expense.education, 2000.0); // Maximum 2000 TL
                description = "Eğitim harcamaları indirimi";
            });
        };

        if (expense.health > 0) {
            deductions.add({
                type_ = "Sağlık";
                amount = expense.health * 0.15; // %15 indirim
                description = "Sağlık harcamaları indirimi";
            });
        };

        if (expense.housing > 0) {
            deductions.add({
                type_ = "Konut";
                amount = Float.min(expense.housing * 0.10, 10000.0);
                description = "Konut kredisi faiz indirimi";
            });
        };

        if (expense.donations > 0) {
            deductions.add({
                type_ = "Bağış";
                amount = expense.donations;
                description = "Bağış indirimi";
            });
        };

        return Buffer.toArray(deductions);
    };

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

    private func generateTaxRecommendations(income: Income, expense: Expense) : [Text] {
        var recommendations = Buffer.Buffer<Text>(0);

        if (income.investment > 0) {
            recommendations.add("Uzun vadeli yatırım araçlarını tercih ederek vergi avantajından yararlanabilirsiniz.");
        };

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

    public func calculateVAT(amount: Float, rate: Float) : async Float {
        return amount * (rate / 100.0);
    };

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

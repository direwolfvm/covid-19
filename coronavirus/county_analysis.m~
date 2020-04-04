
clear
analysis_list_filename = 'DCCountyList.xlsx';
analysis_title = 'DC Metro Area';

%variables

bad_growth_rate = .35;
actual_to_confirmed = 10;
start_date_option = 1; %1 for first data, 2 for day with 20 cases
future_days = 30;
transmission_contacts=10;
hospital_frac=.15;
ICU_frac = .05;
%this tries to correct the bed utilization rate for non-essential occupants
%"kicked out" during the crisis (or not admitted)
essential_bed_frac = .67;


population_override=0;
population=8.63e6;

%import case data
opts = detectImportOptions('us-counties.csv');
%preview('us-counties.csv',opts)

county_data=readtable('us-counties.csv',opts);

%county_data.('serial_date')=datenum(county_data.date);

[records, zzz1] = size(county_data);

%import analysis scope
opts = detectImportOptions(analysis_list_filename);

analysis_list_table = readtable(analysis_list_filename,opts);

[analysis_counties, zzz2] = size(analysis_list_table);

extract_indices = zeros(analysis_counties,records);


opts = detectImportOptions('population.csv');
%preview('us-counties.csv',opts)

population_data=readtable('population.csv',opts);

[numpops, zzz3] = size(population_data);

opts = detectImportOptions('Definitive_Healthcare__USA_Hospital_Beds.csv');
hospital_data=readtable('Definitive_Healthcare__USA_Hospital_Beds.csv',opts);
hospital_data=fillmissing(hospital_data,'constant',0);

[numhosp, zzz4] = size(hospital_data);

for i = 1:analysis_counties
    
    extract_indices(i,:) = (strcmp(analysis_list_table.County_Name(i),county_data.county)&strcmp(analysis_list_table.State(i),county_data.state))';
    
end

if analysis_counties>1
    extract_final = logical(sum(extract_indices));

else
    extract_final = logical(extract_indices);

end
focused_county_data = county_data(extract_final,:);
if population_override~=1
    fipslookup = unique(focused_county_data.fips);
    if analysis_counties>1
        
        fipslookup = repmat(fipslookup',numpops,1);
        population = sum(population_data.Population(logical(sum((population_data.FIPS==fipslookup)'))));
        
    else
        
        population = population_data.Population(population_data.FIPS==fipslookup(1));
        
    end
end
fipslookup = unique(focused_county_data.fips);
fipslookup = repmat(fipslookup',numhosp,1);
hospital_beds = sum(hospital_data.NUM_STAFFED_BEDS(logical(sum((hospital_data.FIPS==fipslookup)'))).*(1-essential_bed_frac*hospital_data.BED_UTILIZATION(logical(sum((hospital_data.FIPS==fipslookup)'))))+hospital_data.Potential_Increase_In_Bed_Capac(logical(sum((hospital_data.FIPS==fipslookup)'))));

ICU_beds = sum(hospital_data.NUM_ICU_BEDS(logical(sum((hospital_data.FIPS==fipslookup)'))).*(1-essential_bed_frac*hospital_data.BED_UTILIZATION(logical(sum((hospital_data.FIPS==fipslookup)')))));




focused_county_data2 = removevars(focused_county_data,{'county','state','fips'});

results = grpstats(focused_county_data2,'date','sum');



results.('serial_date')=datenum(results.date);

if max(results.sum_cases)<10
    start_date_option = 1;
    'Not enough data for this row'
    i
end
if start_date_option == 1
    start_date = results.serial_date(1);
else
    start_date = min(results.serial_date(results.sum_cases>=20));
end

results.('day_index') = results.serial_date-start_date;
results = results(results.day_index>=0,:);

new_cases = results.sum_cases(2:end)-results.sum_cases(1:end-1);
results.('new_cases')=[0;new_cases];
growth_rate = new_cases./results.sum_cases(1:end-1);
results.('growth_rate') = [0;growth_rate];
new_case_change = (results.new_cases(2:end)-results.new_cases(1:end-1))./results.new_cases(1:end-1);
results.('new_case_change')=[0;new_case_change];
results.('actual_cases')=results.sum_cases*actual_to_confirmed;

actual_days = length(results.actual_cases);
projection_days=actual_days+future_days;

mean_growth_rate = mean(growth_rate);

day_index = [0:projection_days-1];


figure(1)
clf


figure1 = figure(1);

% Create axes
axes1 = axes('Parent',figure1);
hold(axes1,'on');

% Create loglog
loglog(movavg(results.sum_cases,'simple',5),movavg(results.new_cases,'simple',6),'LineWidth',4,'Parent',axes1);



% Create ylabel
ylabel('New Cases (Up Bad, Down Good)','HorizontalAlignment','center');

% Create xlabel
xlabel('Confirmed Cases','HorizontalAlignment','center','FontSize',14);

title({sprintf('%s - Are We Winning?',analysis_title)},'HorizontalAlignment','center',...
    'FontWeight','bold');


box(axes1,'on');
hold(axes1,'off');
% Set the remaining axes properties
set(axes1,'XMinorTick','on','XScale','log','YMinorTick','on','YScale','log');
set(axes1,'FontSize',12,'YMinorTick','on','YScale','log','XGrid','on','YGrid','on');

saveas(gcf,sprintf('%s_winning.png',analysis_title))
mlem = gcf; 





growth = fit(results.day_index,results.actual_cases,'exp1');
projection_dates = datetime(day_index+start_date,'ConvertFrom','datenum');

projections = table(day_index',projection_dates');
projections.Properties.VariableNames = {'day_index' 'projection_dates'};

projections.('growth_fit') = min(population,growth.a*exp(growth.b*projections.day_index));

daily_growth = zeros(projection_days,1);
daily_growth(1:actual_days)=results.actual_cases;
bad_daily_growth = zeros(projection_days,1);
bad_daily_growth(1:actual_days)=results.actual_cases;
for j=(actual_days+1):projection_days
    daily_growth(j)=daily_growth(j-1)*(1+mean_growth_rate);
end
projections.('daily_growth')=min(population,daily_growth);

if max(results.actual_cases)<400
    bad_start = actual_days+1;
else
   bad_start = min(results.day_index(results.actual_cases>=400))+2; 
    
end

for j=bad_start:projection_days
    bad_daily_growth(j)=bad_daily_growth(j-1)*(1+bad_growth_rate);
end

projections.('bad_daily_growth')=min(population,bad_daily_growth);


projections.('pop_fraction_gf')=projections.growth_fit/population;
projections.('pop_fraction_dg')=projections.daily_growth/population;
projections.('pop_fraction_bdg')=projections.bad_daily_growth/population;

safe_chance_dg = zeros(projection_days,1);
safe_chance_gf = zeros(projection_days,1);
safe_chance_bdg = zeros(projection_days,1);
safe_chance_gf(1:end-3)=(1-projections.pop_fraction_gf(4:end)).^transmission_contacts;
safe_chance_dg(1:end-3)=(1-projections.pop_fraction_dg(4:end)).^transmission_contacts;
safe_chance_bdg(1:end-3)=(1-projections.pop_fraction_bdg(4:end)).^transmission_contacts;
projections.('safe_chance_dg')=safe_chance_dg;
projections.('safe_chance_gf')=safe_chance_gf;
projections.('safe_chance_bdg')=safe_chance_bdg;

projections.('hospital_beds_dg')=min(1,projections.daily_growth/actual_to_confirmed/hospital_beds*hospital_frac);
projections.('hospital_beds_gf')=min(1,projections.growth_fit/actual_to_confirmed/hospital_beds*hospital_frac);
projections.('hospital_beds_bdg')=min(1,projections.bad_daily_growth/actual_to_confirmed/hospital_beds*hospital_frac);

projections.('ICU_beds_dg')=min(1,projections.daily_growth/actual_to_confirmed/ICU_beds*ICU_frac);
projections.('ICU_beds_gf')=min(1,projections.growth_fit/actual_to_confirmed/ICU_beds*ICU_frac);
projections.('ICU_beds_bdg')=min(1,projections.bad_daily_growth/actual_to_confirmed/ICU_beds*ICU_frac);


figure(2)
clf
% Create figure
figure1 = figure(2);

% Create axes
axes1 = axes('Parent',figure1);
hold(axes1,'on');

% Create multiple lines using matrix input to semilogy
semilogy1 = semilogy(projections.projection_dates,[projections.growth_fit,projections.daily_growth,projections.bad_daily_growth],'LineWidth',2,'Parent',axes1);
set(semilogy1(1),'DisplayName','Projection (Exponential Fit)');
set(semilogy1(2),'DisplayName','Projection (Daily Growth)');
set(semilogy1(3),'DisplayName','Projection (Bad Daily Growth)',...
    'Color',[0 0 0]);
dayplot = plot([datetime('today'),datetime('today')],[1,1e9]);
set(dayplot(1),'DisplayName','Today');
% Create ylabel
ylabel('Projected Cases','HorizontalAlignment','center');

% Create xlabel
xlabel('Date','HorizontalAlignment','center','FontSize',14);

% Create title
title({sprintf('%s - Projected Actual Cases',analysis_title)},'HorizontalAlignment','center',...
    'FontWeight','bold');
ylim(axes1,[10 population+10000]);
hold(axes1,'off');
% Set the remaining axes properties
set(axes1,'FontSize',12,'YMinorTick','on','YScale','log','XGrid','on','YGrid','on');
% Create legend
legend1 = legend(axes1,'show');
set(legend1,'Location','southeast','FontSize',12);

saveas(gcf,sprintf('%s_log_cases_projection.png',analysis_title))

figure(3)
clf
% Create figure
figure1 = figure(3);

% Create axes
axes1 = axes('Parent',figure1);
hold(axes1,'on');

% Create multiple lines using matrix input to semilogy
semilogy1 = plot(projections.projection_dates,[projections.pop_fraction_gf,projections.pop_fraction_dg,projections.pop_fraction_bdg]*100,'LineWidth',2,'Parent',axes1);
set(semilogy1(1),'DisplayName','Projection (Exponential Fit)');
set(semilogy1(2),'DisplayName','Projection (Daily Growth)');
set(semilogy1(3),'DisplayName','Projection (Bad Daily Growth)',...
    'Color',[0 0 0]);
dayplot = plot([datetime('today'),datetime('today')],[0,100]);
set(dayplot(1),'DisplayName','Today');

% Create ylabel
ylabel('Population Fraction (%)','HorizontalAlignment','center');

% Create xlabel
xlabel('Date','HorizontalAlignment','center','FontSize',14);

% Create title
title({sprintf('%s - Projected Population Fraction',analysis_title)},'HorizontalAlignment','center',...
    'FontWeight','bold');
ylim(axes1,[0 20]);
hold(axes1,'off');
% Set the remaining axes properties
set(axes1,'FontSize',12,'YMinorTick','on','XGrid','on','YGrid','on');
% Create legend
legend1 = legend(axes1,'show');
set(legend1,'Location','northwest','FontSize',12);

saveas(gcf,sprintf('%s_population_fraction.png',analysis_title))




figure(4)
clf
% Create figure
figure1 = figure(4);

% Create axes
axes1 = axes('Parent',figure1);
hold(axes1,'on');

% Create multiple lines using matrix input to semilogy
semilogy1 = plot(projections.projection_dates,[projections.safe_chance_gf,projections.safe_chance_dg,projections.safe_chance_bdg]*100,'LineWidth',2,'Parent',axes1);
set(semilogy1(1),'DisplayName','Projection (Exponential Fit)');
set(semilogy1(2),'DisplayName','Projection (Daily Growth)');
set(semilogy1(3),'DisplayName','Projection (Bad Daily Growth)',...
    'Color',[0 0 0]);
dayplot = plot([datetime('today'),datetime('today')],[0,100]);
set(dayplot(1),'DisplayName','Today');

% Create ylabel
ylabel('Safe Chance (%)','HorizontalAlignment','center');

% Create xlabel
xlabel('Date','HorizontalAlignment','center','FontSize',14);

% Create title
title({sprintf('%s - Safe Chance',analysis_title)},'HorizontalAlignment','center',...
    'FontWeight','bold');
ylim(axes1,[0 100]);
hold(axes1,'off');
% Set the remaining axes properties
set(axes1,'FontSize',12,'YMinorTick','on','XGrid','on','YGrid','on');
% Create legend
legend1 = legend(axes1,'show');
set(legend1,'Location','southwest','FontSize',12);

saveas(gcf,sprintf('%s_safety_chance.png',analysis_title))

figure(5)

bar(results.date,results.new_cases)

ylabel('Confirmed New Cases Per Day','HorizontalAlignment','center');

% Create xlabel
xlabel('Date','HorizontalAlignment','center','FontSize',14);

% Create title
title({sprintf('%s - New Cases',analysis_title)},'HorizontalAlignment','center',...
    'FontWeight','bold');

saveas(gcf,sprintf('%s_new_cases.png',analysis_title))



figure(6)
clf
% Create figure
figure1 = figure(6);

% Create axes
axes1 = axes('Parent',figure1);
hold(axes1,'on');

% Create multiple lines using matrix input to semilogy
semilogy1 = plot(projections.projection_dates,[projections.hospital_beds_gf,projections.hospital_beds_dg]*100,'LineWidth',2,'Parent',axes1);
set(semilogy1(1),'DisplayName','Hospital Beds (Exponential Fit)','Color',[0 0 1]);
set(semilogy1(2),'DisplayName','Hospital Beds (Daily Growth)','Color',[1 0 0]);

semilogy1 = plot(projections.projection_dates,[projections.ICU_beds_gf,projections.ICU_beds_dg]*100,'LineWidth',2,'Parent',axes1);
set(semilogy1(1),'DisplayName','ICU Beds (Exponential Fit)','LineStyle','--',...
    'Color',[0 0 1]);
set(semilogy1(2),'DisplayName','ICU Beds (Daily Growth)','LineStyle','--',...
    'Color',[1 0 0]);

dayplot = plot([datetime('today'),datetime('today')],[0,100]);
set(dayplot(1),'DisplayName','Today');

% Create ylabel
ylabel('Bed Capacity (%)','HorizontalAlignment','center');

% Create xlabel
xlabel('Date','HorizontalAlignment','center','FontSize',14);

% Create title
title({sprintf('%s - Hospital Capacity',analysis_title)},'HorizontalAlignment','center',...
    'FontWeight','bold');
ylim(axes1,[0 100]);
hold(axes1,'off');
% Set the remaining axes properties
set(axes1,'FontSize',12,'YMinorTick','on','XGrid','on','YGrid','on');
% Create legend
legend1 = legend(axes1,'show');
set(legend1,'Location','northwest','FontSize',12);

saveas(gcf,sprintf('%s_hospital_capacity.png',analysis_title))



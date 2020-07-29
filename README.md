# Description
This is essentially the same as my [previous project](https://github.com/maxantcliff/football_basic_poisson) on this, but this time I’ve defined the key steps as functions to make it easier to run the code over all seasons that there’s data for at once. Previously, any match involving a newly promoted team was dropped because I didn’t have data on their mean goals from the previous season in the PL. This time, I’ve calculated what the newly promoted teams’ mean goals are across several previous seasons and then used this for each promoted team. 

# Resources
Premier league data is from [here](https://www.football-data.co.uk/englandm.php).

# Method
I apply a very basic version of the [Poisson distribution](https://en.wikipedia.org/wiki/Poisson_distribution). This calculated the probability of an event occuring x times, given its mean frequency.
```python
def poisson(actual, mean):
    return(mean**actual*math.exp(-mean))/math.factorial(actual)
```
The mean frequency of home and away goals is calculated from the previous season for incumbent teams. For newly promoted teams, this isn't possible so I 
calculate the mean frequency for newly promoted teams across a 'back_years' number of previous seasons. 
```python
def promoted_team(betting_year):
    newteaminfo = pd.DataFrame(index=['newteam'],columns=columns)
    newteaminfo[(columns)]=0
    for year in range(betting_year-back_years,betting_year):
        new_season=pd.read_csv('data/EPL/%d.csv'%(year+1),usecols=cols)
        prev_season=pd.read_csv('data/EPL/%d.csv'%(year),usecols=cols)
        newlypromoted = []
        teamlist=[]
        for i in range(len(prev_season)):
            if prev_season.HomeTeam[i] not in teamlist:
                teamlist.append(prev_season.HomeTeam[i])
        for i in range(len(new_season)):
            if new_season.HomeTeam[i] not in newlypromoted:
                if new_season.HomeTeam[i] not in teamlist:
                    newlypromoted.append(new_season.HomeTeam[i])
        for i in range(len(new_season)):
            if new_season.HomeTeam[i] in newlypromoted:
                newteaminfo['home_games']+=1
                newteaminfo['home_goals']+=new_season.FTHG[i]
                newteaminfo['home_conceded']+=new_season.FTAG[i]
            if new_season.AwayTeam[i] in newlypromoted:
                newteaminfo['away_games']+=1
                newteaminfo['away_goals']+=new_season.FTAG[i]
                newteaminfo['away_conceded']+=new_season.FTHG[i]
    newteaminfo['alpha_h']=newteaminfo['home_goals']/newteaminfo['home_games']
    newteaminfo['beta_h']=newteaminfo['home_conceded']/newteaminfo['home_games']
    newteaminfo['alpha_a']=newteaminfo['away_goals']/newteaminfo['away_games']
    newteaminfo['beta_a']=newteaminfo['away_conceded']/newteaminfo['away_games']
    newteaminfo['total_games']=newteaminfo['home_games']+newteaminfo['away_games']
    return(newteaminfo)
```

Make a list of teams from the previous season:
```python
def make_teamlist(data):
    teamlist=[]
    for i in range(len(data)):
        if data.HomeTeam[i] not in teamlist:
            teamlist.append(data.HomeTeam[i])
    teamlist.sort()
    return(teamlist)
```
Then calculate the mean number of home and away goals for each team from last season, and add the 'newteam' data:
```python
def make_teaminfo(data):
    teamlist=make_teamlist(data)
    teaminfo = pd.DataFrame(index=teamlist, columns = columns)
    teaminfo[(columns)]=0
    for i in range(len(data)):
        teaminfo['home_games'][(data.HomeTeam[i])]+=1
        teaminfo['away_games'][(data.AwayTeam[i])]+=1
        teaminfo['home_goals'][(data.HomeTeam[i])]+=data.FTHG[i]
        teaminfo['away_goals'][(data.AwayTeam[i])]+=data.FTAG[i]
        teaminfo['home_conceded'][(data.HomeTeam[i])]+=data.FTAG[i]
        teaminfo['away_conceded'][(data.AwayTeam[i])]+=data.FTHG[i]
    teaminfo['alpha_h']=teaminfo['home_goals']/teaminfo['home_games']
    teaminfo['beta_h']=teaminfo['home_conceded']/teaminfo['home_games']
    teaminfo['alpha_a']=teaminfo['away_goals']/teaminfo['away_games']
    teaminfo['beta_a']=teaminfo['away_conceded']/teaminfo['away_games']
    teaminfo['total_games']=teaminfo['home_games']+teaminfo['away_games']
    newteaminfo=promoted_team(betting_year)
    teaminfo=teaminfo.append(newteaminfo)
    return(teaminfo)
```

If a team is newly promoted for the season that we are betting on, it is replaced with the 'newteam' label:
```python
def newteam_label(season):
    for i in range(len(season)):
        if season.HomeTeam[i] not in teamlist:
            season.HomeTeam[i]='newteam'
        if season.AwayTeam[i] not in teamlist:
            season.AwayTeam[i]='newteam'
    return(season)
```

Use the Poisson distribution to calculate the probability of each score happening up to a maximum number of goals - set here as 10-10. 
Then add the appropriate probabilities together to generate the probability of each W/D/L outcome. 'sum_probs' is just there as a check, and should be close to 1.
```python
def probabilities(season):
    maxscore = 11
    for game in range(len(season)):
            probs=pd.DataFrame(index=range(maxscore**2),columns=['homescore',
                                                         'awayscore','probability'])
            index_counter=0
            for i in range(maxscore):
                for j in range(maxscore):
                    prob = poisson(i, teaminfo['alpha_h'][season.HomeTeam[game]]) * poisson(j,teaminfo['alpha_a'][season.AwayTeam[game]])
                    probs.homescore[index_counter]=i
                    probs.awayscore[index_counter]=j               
                    probs.probability[index_counter]=prob
                    index_counter+=1
            p_win=0
            p_loss=0
            p_draw=0
            for i in range(len(probs)):
                if probs.homescore[i]>probs.awayscore[i]:
                    p_win+=probs.probability[i]
                if probs.homescore[i]<probs.awayscore[i]:
                    p_loss+=probs.probability[i]
                if probs.homescore[i]==probs.awayscore[i]:
                    p_draw+=probs.probability[i] 
            season['p_win'][game]=p_win
            season['p_draw'][game]=p_draw
            season['p_loss'][game]=p_loss
            season['sum_probs'][game]=np.sum((p_win,p_draw,p_loss))
    return(season)
```
The expected value (EV) of a bet (full explanation [here](https://help.smarkets.com/hc/en-gb/articles/214554985-How-to-calculate-expected-value-in-betting)) is in the long run, how much profit you can expect to make per £1 placed bet. For example, if a bookie offered odds of 1.9 on a cointoss (with underlying probability of 50%) you’d start losing money fairly fast if you were to place a lot of bets ([if you're new to betting concepts](https://mybettingsites.co.uk/learn/betting-odds-explained/)). Therefore we want to calculate the EV, and then place bets in accordance.

```python
def expected_value(season):   
    for game in range(len(season)):
        season.ev_win[game]=(season.p_win[game]*(season.B365H[game]-1))-(1-season.p_win[game])
        season.ev_draw[game]=(season.p_draw[game]*(season.B365D[game]-1))-(1-season.p_draw[game])
        season.ev_loss[game]=(season.p_loss[game]*(season.B365A[game]-1))-(1-season.p_loss[game])
    return(season)
```
Now that the betting season's data is ready, we can start placing bets. We want to place a bet on the outcome that has the highest expected value. I also set a 'threshold' 
for the EV, only above which bets are placed. This is to improve the confidence level of bets placed (the higher the EV threshold, in theory the higher the level of confidence in our prediction. 
So the first if/elif branches check this, then the next level of if/else branches update the record of bets and the bankroll according to what the actual outcome is.
The print functions are just there to make sure it's working as hoped - they are silenced for running normally. 

```python

def place_bets(season):
    wager=5
    starting_bankroll=100
    games=len(season) #number of games you want to consider betting on
    bankroll=starting_bankroll
    incorrect=0 # counters for track record
    correct=0
    for game in range(games):
        result=0
        ev_max=max(season.ev_win[game],season.ev_draw[game],season.ev_loss[game]) 
        if season.ev_win[game]==ev_max and season.ev_win[game]>threshold:
            team_bet=season.HomeTeam[game]
            if season.FTHG[game]>season.FTAG[game]:
                betvalue=wager*(season.B365H[game]-1)
                result='won'
                correct+=1
            else:
                betvalue=-wager
                result='lost'
                incorrect+=1
            bankroll+=betvalue
        #print("Bet",season.HomeTeam[game],'vs',season.AwayTeam[game],':backed',team_bet)
        #print('home scored',season.FTHG[game], 'away scored',season.FTAG[game])
        #print('Bet',result)
        #print('Bankroll=',bankroll)
        elif season.ev_draw[game]==ev_max and season.ev_draw[game]>threshold:
            team_bet="draw"
            if season.FTHG[game]==season.FTAG[game]:
                betvalue=wager*(season.B365D[game]-1)
                result='won'
                correct+=1
            else:
                betvalue=-wager
                result='lost'
                incorrect+=1
            bankroll+=betvalue
        #print("Bet",season.HomeTeam[game],'vs',season.AwayTeam[game],':backed',team_bet)
        #print('home scored',season.FTHG[game], 'away scored',season.FTAG[game])
        #print('Bet',result)
        #print('Bankroll=',bankroll)
        elif season.ev_loss[game]==ev_max and season.ev_loss[game]>threshold:
            team_bet=season.AwayTeam[game]
            if season.FTHG[game]<season.FTAG[game]:
                betvalue=wager*(season.B365A[game]-1)
                result='won'
                correct+=1
            else:
                betvalue=-wager
                result='lost'
                incorrect+=1
            bankroll+=betvalue
        #print("Bet",season.HomeTeam[game],'vs',season.AwayTeam[game],':backed',team_bet)
        #print('home scored',season.FTHG[game], 'away scored',season.FTAG[game])
        #print('Bet',result)
        #print('Bankroll=',bankroll)       
    return(wager, starting_bankroll,bankroll, correct, incorrect) 
```

To return the info about the bets placed in a season, I define the following:
```python
def bets_return(wager, starting_bankroll, bankroll, correct,incorrect): 
    betcounter=incorrect+correct
    ROI = ((bankroll - starting_bankroll) /(wager * (betcounter)))
    ROI="{:.2%}".format(ROI)
    print(str(2000+betting_year)+'/'+str(betting_year+1),'season:')
    print(correct,'out of',correct+incorrect,'bets were correct')
    print('ROI=',ROI)
```

Finally, define a your threshold and how many seasons you want to calculate the newly-promoted teams' data for, and run the code across as many seasons as you decide. 
The first available season is a number+back_years to keep the divide between historical seasons and the betting season consistent.
```python

threshold=2
back_years=8
for betting_year in range(2+back_years,19):
        data_year=betting_year-1
        newteaminfo=promoted_team(betting_year)
        data=pd.read_csv('data/EPL/%d.csv'%(data_year),usecols=cols)
        season=pd.read_csv('data/EPL/%d.csv'%(betting_year),usecols=cols)
        data=clean(data)
        season=clean(season)
        teamlist=make_teamlist(data)
        teaminfo=make_teaminfo(data)
        season=full_season_run(season)
        wager, starting_bankroll,bankroll, correct, incorrect = place_bets(season)
        bets_return(wager, starting_bankroll,bankroll, correct, incorrect)
```

# Conclusion

Again, the ROI is pretty jumpy and has a tiny sample size, but I think this is an improvement on my last attempt from a coding perspective! 

Next, I want to build a full Dixon-Coles model and hopefully start building in other factors. 



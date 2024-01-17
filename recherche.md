withHttpOnlyFalse()

http
    .csrf()
        .csrfTokenRepository(new HeaderCsrfTokenRepository())


@RequestMapping("/login")
@PreAuthorize("hasIpAddress('127.0.0.1') or loginAttempts < 3")
public String login() {
    // ...
}


spring.security.user.lockout.enabled=true
spring.security.user.lockout.threshold=3
spring.security.user.lockout.duration=300s


<!--Inclure les dépendances nécessaires dans votre fichier pom.xml pour Spring Boot et les bibliothèques de génération d'OTP. Par exemple, vous pouvez utiliser la bibliothèque jjwt pour la génération de JWT (JSON Web Tokens) qui peuvent être utilisés comme des OTP--!>
<!-- Spring Boot -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- JWT -->
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-api</artifactId>
    <version>0.11.2</version>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-impl</artifactId>
    <version>0.11.2</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.jsonwebtoken</groupId>
    <artifactId>jjwt-jackson</artifactId>
    <version>0.11.2</version>
    <scope>runtime</scope>
</dependency>

<!-- Creér une service qui génerer l'OTP--!>
import io.jsonwebtoken.Jwts;
import io.jsonwebtoken.SignatureAlgorithm;
import org.springframework.stereotype.Service;

import java.util.Date;

@Service
public class OtpService {

    private static final String SECRET_KEY = "votre_clé_secrète";

    public String generateOtp(String username) {
        Date now = new Date();
        Date expiryDate = new Date(now.getTime() + 3600000); // 1 heure d'expiration

        return Jwts.builder()
                .setSubject(username)
                .setIssuedAt(now)
                .setExpiration(expiryDate)
                .signWith(SignatureAlgorithm.HS512, SECRET_KEY)
                .compact();
    }

    public boolean verifyOtp(String otp, String username) {
        try {
            Jwts.parser().setSigningKey(SECRET_KEY).parseClaimsJws(otp);
            // Vérification supplémentaire si nécessaire
            return true;
        } catch (Exception e) {
            return false;
        }
    }
}


<!-- Exposer une API pour générer et vérifier l'OTP--!>
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/otp")
public class OtpController {

    @Autowired
    private OtpService otpService;

    @GetMapping("/generate/{username}")
    public String generateOtp(@PathVariable String username) {
        return otpService.generateOtp(username);
    }

    @GetMapping("/verify/{otp}/{username}")
    public boolean verifyOtp(@PathVariable String otp, @PathVariable String username) {
        return otpService.verifyOtp(otp, username);
    }
}


<!-----------------------------------------------------------------------Côté Backend (Spring Boot)--------------------------------------------------------------------!>

<!--Configure Spring Sécurity pour prendre en charge MFA--!>

@Configuration
@EnableWebSecurity
public class SecurityConfig extends WebSecurityConfigurerAdapter {

    // Autres configurations...

    @Override
    protected void configure(HttpSecurity http) throws Exception {
        http
            // Autres configurations...

            // Activer MFA
            .addFilterBefore(mfaFilter(), UsernamePasswordAuthenticationFilter.class);
    }

    @Bean
    public MfaFilter mfaFilter() {
        return new MfaFilter();
    }
}


<!--Créer un Provider d'Authentification MFA--!>

public class MfaAuthenticationProvider extends AbstractUserDetailsAuthenticationProvider {
    @Override
    protected void additionalAuthenticationChecks(UserDetails userDetails, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
        // Implémentez les vérifications MFA supplémentaires
    }

    @Override
    protected UserDetails retrieveUser(String username, UsernamePasswordAuthenticationToken authentication) throws AuthenticationException {
        // Implémentez la récupération de l'utilisateur avec MFA
    }
}


<!--Configurer le stockage des utilisateurs avec MFA--!>

import javax.persistence.Entity;
import javax.persistence.GeneratedValue;
import javax.persistence.GenerationType;
import javax.persistence.Id;

@Entity
public class User {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    private String username;
    private String password; // Vous devriez stocker le mot de passe de manière sécurisée, comme avec Bcrypt

    private boolean mfaEnabled;

    // Getters et Setters

    public Long getId() {
        return id;
    }

    public void setId(Long id) {
        this.id = id;
    }

    public String getUsername() {
        return username;
    }

    public void setUsername(String username) {
        this.username = username;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }

    public boolean isMfaEnabled() {
        return mfaEnabled;
    }

    public void setMfaEnabled(boolean mfaEnabled) {
        this.mfaEnabled = mfaEnabled;
    }
}


<!-----------------------------------------------------------------------Côté Frontend (Angular)---------------------------------------------------------------------!>

<!--Configurer l'interface utilisateur pour MFA--!>

// mfa.component.ts

import { Component } from '@angular/core';
import { AuthService } from 'chemin-vers-votre-service-auth'; // Assurez-vous de spécifier le chemin correct

@Component({
  selector: 'app-mfa',
  template: `
    <div *ngIf="mfaEnabled">
      <h2>Saisir le deuxième facteur d'authentification</h2>
      <form (ngSubmit)="submitMfaCode()">
        <label for="mfaCode">Code MFA :</label>
        <input type="text" id="mfaCode" name="mfaCode" [(ngModel)]="mfaCode" required>
        <button type="submit">Valider</button>
      </form>
    </div>
    <div *ngIf="!mfaEnabled">
      <p>MFA n'est pas activé pour cet utilisateur.</p>
    </div>
  `,
  styles: [`
    /* Ajoutez des styles selon vos besoins */
  `]
})
export class MfaComponent {

  mfaEnabled: boolean = false;
  mfaCode: string = '';

  constructor(private authService: AuthService) {
    // Vérifier si MFA est activé pour l'utilisateur actuel (vous devez implémenter cela dans votre service d'authentification)
    this.authService.checkMfaStatus().subscribe((result) => {
      this.mfaEnabled = result;
    });
  }

  submitMfaCode() {
    // Envoyer le code MFA au backend pour vérification (vous devez implémenter cela dans votre service d'authentification)
    this.authService.submitMfaCode(this.mfaCode).subscribe((result) => {
      if (result.success) {
        // Rediriger l'utilisateur vers la page appropriée après une authentification réussie
      } else {
        // Gérer les erreurs d'authentification MFA
        console.error(result.message);
      }
    });
  }
}


<!--Envoyer les données MFA au Backend--!>

// Exemple avec Angular HTTP Client
import { HttpClient } from '@angular/common/http';

// ...

submitMfaCode(code: string) {
    const mfaData = { code: code };
    this.http.post('/api/mfa', mfaData).subscribe(response => {
        // Gérer la réponse du serveur
    });
}


<!--Gérer l'État Authentifié/Déconnecté côté Frontend--!>

// mfa.component.ts

import { Component } from '@angular/core';
import { AuthService } from 'chemin-vers-votre-service-auth'; // Assurez-vous de spécifier le chemin correct
import { Router } from '@angular/router';

@Component({
  selector: 'app-mfa',
  template: `
    <div *ngIf="authenticated">
      <h2>Bienvenue, {{ username }}!</h2>
      <!-- Ajoutez le reste de votre interface utilisateur authentifiée ici -->
    </div>
    <div *ngIf="!authenticated && mfaEnabled">
      <h2>Saisir le deuxième facteur d'authentification</h2>
      <form (ngSubmit)="submitMfaCode()">
        <label for="mfaCode">Code MFA :</label>
        <input type="text" id="mfaCode" name="mfaCode" [(ngModel)]="mfaCode" required>
        <button type="submit">Valider</button>
      </form>
    </div>
    <div *ngIf="!authenticated && !mfaEnabled">
      <p>MFA n'est pas activé pour cet utilisateur.</p>
    </div>
    <div *ngIf="error">
      <p class="error-message">{{ error }}</p>
    </div>
  `,
  styles: [`
    /* Ajoutez des styles selon vos besoins */
    .error-message {
      color: red;
    }
  `]

export class MfaComponent {

  authenticated: boolean = false;
  mfaEnabled: boolean = false;
  mfaCode: string = '';
  username: string = ''; // Ajoutez votre logique pour obtenir le nom d'utilisateur

  error: string = '';

  constructor(private authService: AuthService, private router: Router) {
    // Vérifier si MFA est activé pour l'utilisateur actuel (vous devez implémenter cela dans votre service d'authentification)
    this.authService.checkMfaStatus().subscribe((result) => {
      this.mfaEnabled = result;
    });

    // Vérifier si l'utilisateur est authentifié avec MFA
    this.authService.isAuthenticated().subscribe((result) => {
      this.authenticated = result;
      if (result) {
        // Rediriger l'utilisateur vers la page appropriée après une authentification réussie
        this.router.navigate(['/accueil']);
      }
    });
  }

  submitMfaCode() {
    // Envoyer le code MFA au backend pour vérification (vous devez implémenter cela dans votre service d'authentification)
    this.authService.submitMfaCode(this.mfaCode).subscribe((result) => {
      if (result.success) {
        this.authenticated = true;
      } else {
        // Gérer les erreurs d'authentification MFA
        this.error = result.message;
        console.error(result.message);
      }
    });
  }
}
